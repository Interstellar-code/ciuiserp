# Purchases Plugin Documentation

## Overview

The Purchases plugin is a comprehensive procurement management system built for CiuisERP. It provides end-to-end purchase order management, vendor relationships, requisition workflows, and seamless integration with inventory, accounting, and partner management modules.

## Architecture

### Core Components

1. **Purchase Orders** - Main procurement documents
2. **Purchase Requisitions** - Internal purchase requests
3. **Vendors/Partners** - Supplier management
4. **Order Lines** - Individual line items
5. **Purchase Agreements** - Long-term contracts
6. **Quotations** - RFQ/Quote management

### Plugin Structure

```
plugins/webkul/purchases/
├── database/
│   ├── migrations/       # Database schema
│   ├── factories/        # Test data factories
│   └── settings/         # System settings
├── src/
│   ├── Models/           # Eloquent models
│   ├── Enums/            # Business logic enums
│   ├── Filament/         # Admin UI resources
│   ├── Settings/         # Configuration classes
│   ├── Mail/             # Email templates
│   └── Livewire/         # Interactive components
└── resources/
    ├── views/            # Blade templates
    └── lang/             # Translations
```

## Database Schema

### Core Tables

#### `purchases_orders`
Primary purchase order table storing order headers.

**Key Fields:**
- `name` - Auto-generated order number (PO/###)
- `state` - Order workflow state (draft, sent, purchase, done, canceled)
- `invoice_status` - Billing status (no, to_invoiced, invoiced)
- `receipt_status` - Receiving status (no, pending, partial, full)
- `partner_id` - Vendor/supplier reference
- `currency_id` - Transaction currency
- `fiscal_position_id` - Tax handling rules
- `payment_term_id` - Payment terms
- `ordered_at`, `approved_at`, `planned_at` - Workflow timestamps

**Financial Fields:**
- `untaxed_amount` - Subtotal before taxes
- `tax_amount` - Total tax amount
- `total_amount` - Grand total
- `currency_rate` - Exchange rate for multi-currency

#### `purchases_order_lines`
Individual line items for each purchase order.

**Key Fields:**
- `product_id` - Product reference
- `product_qty` - Ordered quantity
- `price_unit` - Unit price
- `discount` - Line discount percentage
- `price_subtotal`, `price_total` - Calculated amounts
- `qty_received`, `qty_invoiced` - Tracking received/billed quantities
- `planned_at` - Expected delivery date

#### `purchases_requisitions`
Internal purchase requests that can generate purchase orders.

**Key Fields:**
- `type` - Requisition type (purchase_tender, blanket_order)
- `state` - Workflow state (draft, confirmed, closed, canceled)
- `partner_id` - Preferred vendor
- `starts_at`, `ends_at` - Validity period

#### `purchases_order_groups`
Grouping mechanism for related purchase orders.

### Integration Tables

#### `purchases_order_account_moves`
Links purchase orders to accounting entries for billing.

#### `purchases_order_operations`
Links purchase orders to inventory operations for receiving.

#### `purchases_order_line_taxes`
Many-to-many relationship for tax application on order lines.

## Business Logic

### Order Workflow States

#### Order States (`OrderState` enum)
- **DRAFT** - Initial state, order being prepared
- **SENT** - RFQ sent to vendors, awaiting responses
- **PURCHASE** - Confirmed order, awaiting delivery
- **DONE** - Completed/locked order
- **CANCELED** - Canceled order

#### Invoice Status (`OrderInvoiceStatus` enum)
- **NO** - No billing required/generated
- **TO_INVOICED** - Ready for billing
- **INVOICED** - Fully billed

#### Receipt Status (`OrderReceiptStatus` enum)
- **NO** - No receiving required/generated
- **PENDING** - Awaiting delivery
- **PARTIAL** - Partially received
- **FULL** - Fully received

### Core Business Operations

#### Purchase Order Lifecycle

1. **Creation** - Draft order with line items
2. **RFQ Process** - Send Request for Quotation to vendors
3. **Confirmation** - Convert to confirmed purchase order
4. **Receiving** - Create inventory receipts
5. **Billing** - Generate vendor bills
6. **Completion** - Order fulfillment

#### Key Methods (`PurchaseOrder` class)

```php
// Send RFQ to vendors
public function sendRFQ(Order $record, array $data): Order

// Confirm purchase order
public function confirmPurchaseOrder(Order $record): Order

// Send confirmed PO to vendor
public function sendPurchaseOrder(Order $record, array $data): Order

// Create vendor bill
public function createPurchaseOrderBill(Order $record): Order

// Cancel order and inventory operations
public function cancelPurchaseOrder(Order $record): Order
```

## Entity Relationships

### Purchase Orders Integration

#### With Partners (Vendors)
- **Vendor Management** - Supplier information, payment terms, contacts
- **Vendor Prices** - Product-specific pricing by vendor
- **Purchase History** - Track vendor performance

#### With Products
- **Product Catalog** - Purchasable products and services
- **Vendor Information** - Supplier-specific product details
- **Unit of Measure** - Purchase UOM vs. stock UOM conversion

#### With Inventory
- **Receipt Creation** - Auto-generate inventory receipts
- **Stock Moves** - Track product movements
- **Location Management** - Destination warehouses/locations
- **Serial/Lot Tracking** - Traceability for received goods

#### With Accounting
- **Vendor Bills** - Generate accounting entries
- **Tax Calculation** - Apply vendor-specific tax rules
- **Multi-Currency** - Handle foreign vendors
- **Cost Accounting** - Product cost updates

### Data Flow Example

```
Requisition → Purchase Order → Inventory Receipt → Vendor Bill
     ↓              ↓              ↓              ↓
   Approval     RFQ/Confirm    Stock Update   Account Entry
```

## Configuration & Settings

### Order Settings (`OrderSettings`)
- `enable_order_approval` - Require approval for orders
- `order_validation_amount` - Approval threshold amount
- `enable_lock_confirmed_orders` - Lock orders after confirmation
- `enable_purchase_agreements` - Enable agreement functionality

### Product Settings (`ProductSettings`)
- Purchase-specific product configuration
- Vendor relationship settings
- Cost calculation methods

## API Endpoints

### Web Routes

#### Public Quotation Response
```
GET /purchase/{order}/{action}
```
- Signed route for vendor quotation responses
- Actions: accept, decline, modify
- Secure vendor portal access

### Recommended API Structure

#### Purchase Orders
```
GET    /api/purchase-orders           # List orders
POST   /api/purchase-orders           # Create order
GET    /api/purchase-orders/{id}      # View order
PUT    /api/purchase-orders/{id}      # Update order
DELETE /api/purchase-orders/{id}      # Cancel order

POST   /api/purchase-orders/{id}/confirm     # Confirm order
POST   /api/purchase-orders/{id}/send-rfq    # Send RFQ
POST   /api/purchase-orders/{id}/create-bill # Create bill
```

#### Vendors
```
GET    /api/vendors                   # List vendors
POST   /api/vendors                   # Create vendor
GET    /api/vendors/{id}              # View vendor
PUT    /api/vendors/{id}              # Update vendor
GET    /api/vendors/{id}/orders       # Vendor's orders
GET    /api/vendors/{id}/products     # Vendor's products
```

#### Requisitions
```
GET    /api/requisitions              # List requisitions
POST   /api/requisitions              # Create requisition
GET    /api/requisitions/{id}         # View requisition
PUT    /api/requisitions/{id}         # Update requisition
POST   /api/requisitions/{id}/convert # Convert to PO
```

## Key Features

### Purchase Order Management
- **Multi-step Workflow** - Draft → RFQ → Confirmed → Delivered
- **Approval Process** - Configurable approval thresholds
- **Document Generation** - PDF quotations and purchase orders
- **Email Integration** - Automated vendor communications

### Vendor Management
- **Vendor Portal** - Quotation response interface
- **Price Lists** - Vendor-specific product pricing
- **Payment Terms** - Vendor-specific payment conditions
- **Performance Tracking** - Delivery and quality metrics

### Inventory Integration
- **Automatic Receipts** - Generate receipts from confirmed orders
- **Three-way Matching** - PO → Receipt → Invoice matching
- **Serial/Lot Tracking** - Full traceability
- **Multiple Locations** - Support for multiple warehouses

### Financial Integration
- **Multi-Currency** - Handle international vendors
- **Tax Management** - Vendor-specific tax rules
- **Cost Accounting** - Update product costs from purchases
- **Budgeting** - Purchase budget tracking and controls

### Reporting & Analytics
- **Purchase Analytics** - Spending analysis by vendor, category
- **Vendor Performance** - Delivery time, quality metrics
- **Cost Trends** - Price history and trends
- **Compliance** - Audit trails and document management

## Advanced Features

### Purchase Agreements
- **Blanket Orders** - Long-term purchasing contracts
- **Volume Discounts** - Quantity-based pricing tiers
- **Contract Management** - Terms and conditions tracking

### Requisition Management
- **Internal Requests** - Department-based purchase requests
- **Approval Workflows** - Multi-level approvals
- **Budget Controls** - Department budget enforcement

### Automation
- **Reorder Points** - Automatic purchase suggestions
- **Vendor Selection** - Best price/vendor algorithms
- **Receipt Matching** - Automated three-way matching

## Technical Implementation

### Key Models

#### `Order` Model
- Core purchase order entity
- Integrates with Chatter, Custom Fields, Activity Logging
- Automatic name generation (PO/###)
- State management and workflow

#### `OrderLine` Model
- Sortable line items
- Tax calculation integration
- Inventory move tracking
- Account move line linking

#### `Requisition` Model
- Internal purchase requests
- Conversion to purchase orders
- Approval workflow support

### Facades & Managers

#### `PurchaseOrder` Facade
```php
use Webkul\Purchase\Facades\PurchaseOrder;

// Send RFQ
PurchaseOrder::sendRFQ($order, $emailData);

// Confirm order
PurchaseOrder::confirmPurchaseOrder($order);

// Create bill
PurchaseOrder::createPurchaseOrderBill($order);
```

### Events & Observers

#### Account Move Observer
- Automatic accounting entry creation
- Integration with financial workflows
- Multi-currency handling

## Security & Permissions

### Policy Classes
- `PurchaseOrderPolicy` - Order access control
- `VendorPolicy` - Vendor management permissions
- `RequisitionPolicy` - Requisition workflow permissions

### User Roles
- **Purchase Manager** - Full procurement access
- **Purchase User** - Create/manage assigned orders
- **Requestor** - Create requisitions only
- **Vendor** - Limited portal access

## Customization

### Custom Fields
- Extensible through Field plugin
- Order and line item customization
- Vendor-specific fields

### Workflow Customization
- Configurable approval processes
- Custom order states
- Business rule integration

### Integration Points
- **ERP Systems** - Standard API interfaces
- **E-commerce** - Dropshipping integration
- **Supplier Portals** - EDI/API connections
- **Financial Systems** - Accounting integration

## Best Practices

### Order Management
1. **Standardize Processes** - Consistent ordering workflows
2. **Vendor Qualification** - Proper vendor onboarding
3. **Price Validation** - Regular price list updates
4. **Approval Controls** - Appropriate authorization levels

### Data Management
1. **Master Data** - Clean vendor and product data
2. **Cost Tracking** - Accurate cost accounting
3. **Document Management** - Proper filing and retention
4. **Audit Trails** - Complete transaction history

### Performance Optimization
1. **Bulk Operations** - Efficient mass updates
2. **Caching Strategy** - Price list and vendor data
3. **Queue Management** - Background email processing
4. **Database Indexing** - Optimized queries

## Troubleshooting

### Common Issues

#### Order State Problems
- **Stuck in Draft** - Check required fields and validations
- **Cannot Confirm** - Verify approval settings and user permissions
- **Receipt Issues** - Ensure inventory plugin is installed and configured

#### Integration Issues
- **Accounting Errors** - Verify chart of accounts and tax settings
- **Inventory Problems** - Check warehouse and location configuration
- **Email Failures** - Validate mail settings and vendor email addresses

#### Performance Issues
- **Slow Queries** - Check database indexes and query optimization
- **Memory Usage** - Review large order processing and bulk operations
- **Email Queues** - Monitor queue processing for vendor communications

## Migration & Deployment

### Database Migrations
- Sequential migration files ensure proper schema evolution
- Foreign key constraints maintain data integrity
- Conditional migrations support plugin dependencies

### Settings Migration
- Default configuration for new installations
- Upgrade paths for existing systems
- Environment-specific settings

This documentation provides a comprehensive overview of the Purchases plugin's functionality, architecture, and integration capabilities within the CiuisERP ecosystem.