# Accounts Plugin Documentation

The Accounts plugin is the core financial management system for CiuisERP, implementing sophisticated double-entry accounting, tax calculation engines, payment processing, and financial reconciliation workflows.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Database Schema](#database-schema)
4. [Core Business Logic](#core-business-logic)
5. [Tax Engine](#tax-engine)
6. [Payment System](#payment-system)
7. [Reconciliation System](#reconciliation-system)
8. [Integration Points](#integration-points)
9. [API Reference](#api-reference)
10. [Configuration](#configuration)

## Overview

The Accounts plugin implements enterprise-grade financial accounting features:

- **Double-Entry Accounting**: Complete chart of accounts with journal entries
- **Multi-Currency Support**: Currency-aware transactions and reporting
- **Advanced Tax Calculation**: Complex tax scenarios with cascading calculations
- **Invoice Management**: Customer invoices, vendor bills, credit notes, refunds
- **Payment Processing**: Multiple payment methods and reconciliation
- **Financial Reconciliation**: Bank statement matching and outstanding balances
- **Fiscal Positions**: Country-specific tax handling and compliance
- **Multi-Company**: Isolated accounting per company entity

## Architecture

### Core Components

```
plugins/webkul/accounts/
├── src/
│   ├── AccountManager.php          # Core accounting business logic
│   ├── TaxManager.php             # Tax calculation engine
│   ├── AccountServiceProvider.php  # Plugin service registration
│   ├── AccountPlugin.php          # Filament plugin integration
│   ├── Models/                    # Eloquent models for financial data
│   ├── Enums/                     # Business constants and states
│   ├── Facades/                   # Service facades
│   ├── Filament/                  # Admin UI resources
│   ├── Livewire/                  # Real-time UI components
│   └── Traits/                    # Reusable model behaviors
├── database/
│   ├── migrations/                # 40+ database tables
│   ├── factories/                 # Test data factories
│   └── seeders/                   # Initial data seeders
└── resources/
    ├── views/                     # Blade templates
    └── lang/                      # Internationalization
```

### Service Architecture

The plugin registers two core services via facades:

- **Account Facade**: Access to `AccountManager` for move-based operations
- **Tax Facade**: Access to `TaxManager` for tax calculations

### Dependencies

- **Products Plugin**: For product-based invoicing
- **Partners Plugin**: Customer and vendor management integration
- **Support Plugin**: Core framework services

## Database Schema

### Core Financial Tables

#### 1. Chart of Accounts (`accounts_accounts`)

```sql
-- Core account structure following double-entry principles
- id (primary key)
- account_type (enum: asset_receivable, asset_cash, liability_payable, etc.)
- name (account name)
- code (account code)
- currency_id (foreign key)
- reconcile (boolean: whether account requires reconciliation)
- deprecated (boolean)
- non_trade (boolean)
```

**Account Types Supported:**
- **Assets**: receivable, cash, current, non_current, prepayments, fixed
- **Liabilities**: payable, credit_card, current, non_current
- **Equity**: equity, equity_unaffected
- **Income**: income, income_other  
- **Expenses**: expense, expense_depreciation, expense_direct_cost
- **Off-Balance**: off_balance

#### 2. Journal Entries (`accounts_account_moves`)

```sql
-- Central transaction record in move-based accounting
- id (primary key)
- journal_id (foreign key to accounts_journals)
- company_id (multi-company support)
- partner_id (customer/vendor)
- state (enum: draft, posted, cancel)
- move_type (enum: out_invoice, in_invoice, out_refund, in_refund, entry)
- currency_id (multi-currency)
- payment_state (enum: not_paid, in_payment, paid, partial, reversed, invoicing_legacy)
- amount_untaxed (decimal 15,4)
- amount_tax (decimal 15,4)
- amount_total (decimal 15,4)
- amount_residual (decimal 15,4)
- invoice_date_due (payment due date)
- fiscal_position_id (tax handling rules)
- invoice_payment_term_id (payment terms)
```

**Move Types:**
- `out_invoice`: Customer Invoice
- `in_invoice`: Vendor Bill  
- `out_refund`: Customer Credit Note
- `in_refund`: Vendor Credit Note
- `entry`: Manual Journal Entry
- `out_receipt`: Customer Receipt
- `in_receipt`: Vendor Receipt

#### 3. Journal Entry Lines (`accounts_account_move_lines`)

```sql
-- Individual debit/credit lines within journal entries
- id (primary key)
- move_id (foreign key to accounts_account_moves)
- account_id (chart of accounts reference)
- partner_id (customer/vendor for receivables/payables)
- product_id (product reference for invoice lines)
- display_type (enum: product, tax, payment_term)
- debit (decimal 15,4)
- credit (decimal 15,4)
- balance (decimal 15,4: debit - credit)
- amount_currency (multi-currency amount)
- quantity (product quantity)
- price_unit (unit price)
- price_subtotal (line subtotal)
- price_total (line total with tax)
- discount (percentage)
- tax_base_amount (base for tax calculation)
- reconcile_id (reconciliation tracking)
- full_reconcile_id (complete reconciliation)
```

#### 4. Tax System (`accounts_taxes`)

```sql
-- Comprehensive tax configuration
- id (primary key)
- company_id (company-specific taxes)
- tax_group_id (tax categorization)
- type_tax_use (enum: sale, purchase, none)
- tax_scope (enum: service, consu)
- amount_type (enum: percent, fixed)
- amount (tax rate/amount)
- price_include_override (tax_included, tax_excluded)
- include_base_amount (boolean: affects other tax bases)
- is_base_affected (boolean: affected by other taxes)
- analytic (boolean: analytic accounting)
```

#### 5. Payment System (`accounts_account_payments`)

```sql
-- Payment transactions and reconciliation
- id (primary key)
- move_id (associated journal entry)
- partner_id (payer/payee)
- payment_method_line_id (payment method)
- currency_id (payment currency)
- payment_type (enum: outbound, inbound, transfer)
- partner_type (enum: customer, supplier)
- amount (payment amount)
- state (enum: draft, posted, sent, reconciled, cancelled)
- is_reconciled (boolean)
- date (payment date)
```

### Advanced Financial Features

#### Fiscal Positions (`accounts_fiscal_positions`)
Tax mapping rules for different regions/countries with automatic tax substitution.

#### Cash Rounding (`accounts_cash_roundings`)
Configurable rounding strategies for cash payments with accounting impact.

#### Payment Terms (`accounts_payment_terms`)
Flexible payment term definitions with multiple due date calculations.

#### Bank Reconciliation
- `accounts_bank_statements`: Bank statement imports
- `accounts_bank_statement_lines`: Individual bank transactions
- `accounts_reconciles`: Partial reconciliation tracking
- `accounts_full_reconciles`: Complete reconciliation records

## Core Business Logic

### AccountManager

The `AccountManager` class orchestrates all financial operations:

#### Key Methods

```php
// Journal entry state management
public function confirm(AccountMove $record): AccountMove
public function cancel(AccountMove $record): AccountMove  
public function resetToDraft(AccountMove $record): AccountMove

// Core computation engine
public function computeAccountMove(AccountMove $record): AccountMove
public function computeMoveLine(AccountMove $move, MoveLine $line): MoveLine
public function computeMoveLineTotals(MoveLine $line, array &$newTaxEntries): array

// Communication
public function printAndSend(AccountMove $record, array $data): AccountMove
```

#### Move-Based Accounting Workflow

1. **Draft Creation**: Invoice/bill created in draft state
2. **Line Computation**: Product lines calculated with taxes
3. **Move Computation**: Journal entry totals computed
4. **Tax Line Generation**: Automatic tax journal lines created
5. **Payment Term Line**: Receivable/payable line generated
6. **Confirmation**: Move posted to ledger (immutable)

### Move Computation Engine

```php
public function computeAccountMove(AccountMove $record): AccountMove
{
    // Reset all computed amounts
    $record->amount_untaxed = 0;
    $record->amount_tax = 0;
    $record->amount_total = 0;
    
    // Process each product line
    foreach ($record->lines as $line) {
        $line = $this->computeMoveLine($record, $line);
        [$line, $amountTax] = $this->computeMoveLineTotals($line, $newTaxEntries);
        
        // Accumulate totals
        $record->amount_untaxed += $line->price_subtotal;
        $record->amount_tax += $amountTax;
        $record->amount_total += $line->price_total;
    }
    
    // Generate tax and payment term lines
    $this->computeTaxLines($record, $newTaxEntries);
    $this->computePaymentTermLine($record);
    
    return $record;
}
```

### Document Type Detection

```php
// Inbound vs Outbound detection
public function isInbound($includeReceipts = true)
{
    return in_array($this->move_type, [
        MoveType::OUT_INVOICE,  // Customer invoice (money coming in)
        MoveType::IN_REFUND     // Vendor refund (money coming in)
    ]);
}

public function isOutbound($includeReceipts = true)  
{
    return in_array($this->move_type, [
        MoveType::IN_INVOICE,   // Vendor bill (money going out)
        MoveType::OUT_REFUND    // Customer refund (money going out)
    ]);
}
```

## Tax Engine

### TaxManager

The `TaxManager` provides sophisticated tax calculation supporting:

- **Cascade Taxation**: Taxes affecting base amounts of subsequent taxes
- **Tax Inclusion**: Price-inclusive vs price-exclusive scenarios
- **Multi-Tax Application**: Multiple taxes per line item
- **Currency Handling**: Tax calculations in multiple currencies

#### Tax Calculation Algorithm

```php
public static function collect($taxIds, $subTotal, $quantity)
{
    $taxes = Tax::whereIn('id', $taxIds)->orderBy('sort')->get();
    $totalTaxAmount = 0;
    $adjustedSubTotal = $subTotal;
    
    foreach ($taxes as $tax) {
        $currentTaxBase = $adjustedSubTotal;
        
        // Handle cascade taxation
        if ($tax->is_base_affected) {
            foreach ($taxesComputed as $prevTax) {
                if ($prevTax['include_base_amount']) {
                    $currentTaxBase += $prevTax['tax_amount'];
                }
            }
        }
        
        // Calculate tax amount
        if ($tax->price_include_override == 'tax_included') {
            // Back-calculate from inclusive price
            if ($tax->amount_type == 'percent') {
                $taxFactor = $amount / 100;
                $currentTaxAmount = $currentTaxBase - ($currentTaxBase / (1 + $taxFactor));
            }
            $adjustedSubTotal -= $currentTaxAmount;
        } else {
            // Forward calculation from exclusive price
            if ($tax->amount_type == 'percent') {
                $currentTaxAmount = $currentTaxBase * $amount / 100;
            } else {
                $currentTaxAmount = $amount * $quantity;
            }
        }
        
        $totalTaxAmount += $currentTaxAmount;
    }
    
    return [$adjustedSubTotal, $totalTaxAmount, $taxesComputed];
}
```

### Tax Configuration Types

#### Amount Types
- **Percent**: Percentage-based taxation (e.g., 21% VAT)
- **Fixed**: Fixed amount per unit (e.g., $5 excise tax)

#### Tax Use Categories
- **Sale**: Applied to customer transactions
- **Purchase**: Applied to vendor transactions  
- **None**: No automatic application

#### Price Inclusion
- **tax_included**: Prices include tax (B2C scenarios)
- **tax_excluded**: Prices exclude tax (B2B scenarios)

## Payment System

### Payment Lifecycle

1. **Payment Creation**: Link to invoices/bills being paid
2. **Method Selection**: Choose from configured payment methods
3. **Currency Handling**: Multi-currency payment support
4. **Journal Generation**: Automatic accounting entries
5. **Reconciliation**: Match against bank statements

### Payment Methods and Lines

Payment methods define the framework, while payment method lines provide journal-specific implementations:

```php
// Journal automatically creates method lines for bank/cash journals
public function computeInboundPaymentMethodLines(): void
{
    if (!in_array($this->type, ['bank', 'cash', 'credit'])) {
        return;
    }
    
    $defaultMethods = $this->getDefaultInboundPaymentMethods();
    
    foreach ($defaultMethods as $method) {
        $this->inboundPaymentMethodLines()->create([
            'name' => $method->name,
            'payment_method_id' => $method->id,
            'type' => 'inbound',
        ]);
    }
}
```

## Reconciliation System

### Bank Statement Reconciliation

1. **Statement Import**: Bank statements loaded into system
2. **Statement Lines**: Individual transactions identified  
3. **Matching Engine**: Automatic matching against outstanding invoices
4. **Manual Reconciliation**: User intervention for unmatched items
5. **Full Reconciliation**: Complete closure of accounting items

### Reconciliation Types

#### Partial Reconciles (`accounts_partial_reconciles`)
Temporary reconciliation groupings for complex matching scenarios.

#### Full Reconciles (`accounts_full_reconciles`) 
Complete reconciliation of accounting items with balanced debits and credits.

### Outstanding Balance Management

The system tracks outstanding balances on move lines:
- `amount_residual`: Remaining amount to be paid
- `amount_residual_currency`: Residual in original currency
- `reconciled`: Boolean flag for fully reconciled items

## Integration Points

### Partner Plugin Integration

The accounts plugin extends the partner model with financial attributes:

```sql
-- Added to partners_partners table
- property_account_payable_id (default payable account)
- property_account_receivable_id (default receivable account)  
- property_payment_term_id (default payment terms)
- property_supplier_payment_term_id (vendor payment terms)
- invoice_warning (customer credit warnings)
- credit_limit (customer credit limits)
- debit_limit (vendor limits)
```

### Product Plugin Integration

Products inherit tax configuration:
- `accounts_product_taxes`: Customer tax mapping
- `accounts_product_supplier_taxes`: Vendor tax mapping

### Multi-Company Architecture

All financial data is company-isolated:
- Journal entries scoped by `company_id`
- Chart of accounts per company
- Currency and tax configuration per company
- Cross-company transactions supported via inter-company accounts

## API Reference

### AccountManager API

```php
// Core move operations
Account::computeAccountMove(Move $record): Move
Account::confirm(Move $record): Move
Account::cancel(Move $record): Move
Account::resetToDraft(Move $record): Move

// Communication
Account::printAndSend(Move $record, array $data): Move
```

### TaxManager API

```php
// Tax calculation
Tax::collect(array $taxIds, float $subTotal, float $quantity): array
// Returns: [adjustedSubTotal, totalTaxAmount, taxesComputed]
```

### Model Relationships

#### Move Model
```php
$move->lines()          // Product lines only
$move->allLines()       // All lines including tax/payment
$move->taxLines()       // Tax lines only  
$move->paymentTermLine() // Receivable/payable line
$move->partner()        // Customer/vendor
$move->journal()        // Accounting journal
$move->currency()       // Transaction currency
```

#### MoveLine Model
```php
$line->move()           // Parent journal entry
$line->account()        // Chart of accounts
$line->product()        // Product reference
$line->taxes()          // Applied taxes
$line->partner()        // Line-level partner
```

## Configuration

### Journal Configuration

Journals define the accounting framework:

```php
// Journal types
JournalType::SALE        // Customer invoicing
JournalType::PURCHASE    // Vendor bills
JournalType::BANK        // Bank transactions
JournalType::CASH        // Cash transactions  
JournalType::CREDIT_CARD // Credit card processing
JournalType::GENERAL     // Manual entries
```

### Account Type Configuration

Chart of accounts follows standard accounting categories:

```php
// Asset accounts
AccountType::ASSET_RECEIVABLE    // Customer receivables
AccountType::ASSET_CASH          // Cash accounts
AccountType::ASSET_CURRENT       // Current assets
AccountType::ASSET_FIXED         // Fixed assets

// Liability accounts  
AccountType::LIABILITY_PAYABLE   // Vendor payables
AccountType::LIABILITY_CURRENT   // Current liabilities

// Income/Expense
AccountType::INCOME              // Revenue accounts
AccountType::EXPENSE             // Expense accounts
```

### Currency Configuration

Multi-currency support with:
- Base company currency per entity
- Foreign currency transactions
- Automatic currency rate application
- Currency gain/loss accounting

### Security and Permissions

The plugin integrates with the security framework:
- Role-based access control for financial data
- Resource-level permissions (create, edit, view, delete)
- Company-level data isolation
- Audit trails for all financial transactions

### Performance Considerations

- **Database Indexing**: Optimized indexes on frequently queried fields
- **Lazy Loading**: Relationship data loaded on demand
- **Caching**: Tax calculations cached for performance
- **Bulk Operations**: Batch processing for large datasets

## Advanced Features

### Fiscal Positions

Automatic tax mapping based on customer/vendor location:
- Country-specific tax rules
- Automatic tax substitution
- B2B vs B2C tax handling
- EU VAT compliance

### Cash Rounding

Configurable cash rounding strategies:
- Rounding precision (0.05, 0.10, etc.)
- Rounding method (up, down, half-up)
- Accounting impact via rounding accounts

### Payment Terms

Flexible payment term calculations:
- Net days (e.g., Net 30)
- End of month terms
- Early payment discounts
- Installment payments

### Analytic Accounting

Integration points for cost center and project accounting:
- Move line analytic distribution
- Project-based financial tracking
- Department cost allocation

This documentation provides a comprehensive overview of the Accounts plugin's sophisticated financial management capabilities, from basic invoicing to complex multi-currency, multi-company accounting scenarios.