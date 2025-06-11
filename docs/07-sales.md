# Sales Plugin Documentation

## Overview

The Sales plugin is a comprehensive sales order management system that handles the complete sales lifecycle from quotations to delivery and invoicing. It integrates deeply with the Partners (customers), Products, Inventories, and Accounts modules to provide a unified sales management experience.

## Database Schema

### Core Sales Tables

#### 1. Sales Teams (`sales_teams`)
**Purpose**: Manages sales teams and their performance targets
```sql
- id (Primary Key)
- sort (Integer) - Display order
- company_id (FK to companies)
- user_id (FK to users) - Team Leader
- creator_id (FK to users) - Created by
- name (String) - Team name
- color (String) - UI color identifier
- is_active (Boolean) - Active status
- invoiced_target (Decimal 15,4) - Revenue target
- created_at, updated_at, deleted_at
```

#### 2. Sales Orders (`sales_orders`)
**Purpose**: Core sales order/quotation entity
```sql
- id (Primary Key)
- utm_source_id (FK to utm_sources) - Marketing attribution
- campaign_id (FK to utm_campaigns) - Marketing campaign
- medium_id (FK to utm_mediums) - Marketing medium
- company_id (FK to companies) - Owning company
- partner_id (FK to partners_partners) - Customer
- journal_id (FK to accounts_journals) - Invoicing journal
- partner_invoice_id (FK to partners_partners) - Invoice address
- partner_shipping_id (FK to partners_partners) - Shipping address
- fiscal_position_id (FK to accounts_fiscal_positions) - Tax handling
- payment_term_id (FK to accounts_payment_terms) - Payment terms
- currency_id (FK to currencies) - Order currency
- user_id (FK to users) - Salesperson
- team_id (FK to sales_teams) - Sales team
- creator_id (FK to users) - Created by
- sale_order_template_id (FK to sales_order_templates) - Template used
- warehouse_id (FK to inventories_warehouses) - Fulfillment warehouse
- access_token (String) - Public access token
- name (String) - Order reference (auto-generated)
- state (Enum) - Order state: draft, sent, sale, cancel
- client_order_ref (String) - Customer reference
- origin (String) - Source document
- reference (String) - Payment reference
- signed_by (String) - Digital signature
- invoice_status (Enum) - Invoicing status
- delivery_status (Enum) - Delivery status
- validity_date (Date) - Quotation expiration
- note (Text) - Terms and conditions
- currency_rate (Decimal 15,4) - Exchange rate
- amount_untaxed (Decimal 15,4) - Subtotal
- amount_tax (Decimal 15,4) - Total tax
- amount_total (Decimal 15,4) - Grand total
- locked (Boolean) - Edit locked
- require_signature (Boolean) - Requires signature
- require_payment (Boolean) - Requires payment
- commitment_date (Date) - Delivery commitment
- date_order (Date) - Order date
- signed_on (Date) - Signature date
- prepayment_percent (Decimal 15,4) - Down payment percentage
```

#### 3. Sales Order Lines (`sales_order_lines`)
**Purpose**: Individual line items within sales orders
```sql
- id (Primary Key)
- sort (Integer) - Line order
- order_id (FK to sales_orders) - Parent order
- company_id (FK to companies)
- currency_id (FK to currencies)
- order_partner_id (FK to partners_partners) - Customer
- salesman_id (FK to users) - Salesperson
- product_id (FK to products_products) - Product
- product_uom_id (FK to unit_of_measures) - Unit of measure
- linked_sale_order_sale_id (FK to sales_order_lines) - Linked line
- product_packaging_id (FK to products_packagings) - Packaging
- creator_id (FK to users) - Created by
- warehouse_id (FK to inventories_warehouses) - Warehouse
- state (Enum) - Line state
- display_type (Enum) - Display type (line_section, line_note, etc.)
- virtual_id (String) - Virtual identifier
- linked_virtual_id (String) - Linked virtual ID
- qty_delivered_method (Enum) - Delivery method (manual, stock_move)
- invoice_status (Enum) - Invoice status
- analytic_distribution (String) - Analytics distribution
- name (String) - Line description
- product_uom_qty (Decimal 15,4) - Ordered quantity
- product_qty (Decimal 15,4) - Product quantity
- price_unit (Decimal 15,4) - Unit price
- discount (Decimal 15,4) - Discount percentage
- price_subtotal (Decimal 15,4) - Line subtotal
- price_total (Decimal 15,4) - Line total with tax
- price_reduce_taxexcl (Decimal 15,4) - Reduced price excl. tax
- price_reduce_taxinc (Decimal 15,4) - Reduced price incl. tax
- purchase_price (Decimal 15,4) - Cost price
- margin (Decimal 15,4) - Profit margin
- margin_percent (Decimal 15,4) - Margin percentage
- qty_delivered (Decimal 15,4) - Delivered quantity
- qty_invoiced (Decimal 15,4) - Invoiced quantity
- qty_to_invoice (Decimal 15,4) - Quantity to invoice
- untaxed_amount_invoiced (Decimal 15,4) - Invoiced amount
- untaxed_amount_to_invoice (Decimal 15,4) - Amount to invoice
- is_downpayment (Boolean) - Down payment line
- is_expense (Boolean) - Expense line
- technical_price_unit (Decimal 15,4) - Technical unit price
- price_tax (Decimal 15,4) - Tax amount
- product_packaging_qty (Decimal 15,4) - Packaging quantity
- customer_lead (Decimal 15,4) - Lead time in days
```

## Business Logic & Entity Relationships

### Sales Manager (`Webkul\Sale\SaleManager`)

The SaleManager handles core business logic:

#### Order State Transitions
- **Quotation to Sale Order**: Confirms orders and triggers inventory operations
- **Back to Quotation**: Reverts confirmed orders back to draft
- **Cancel Order**: Cancels orders and associated inventory moves

#### Financial Computations
- **Order-Level Calculations**: Computes totals from line items
- **Line-Level Calculations**: Calculates subtotals, discounts, and taxes
- **Margin Analysis**: Calculates profit margins per line

#### Inventory Integration
- **Pull Rule Application**: Creates stock moves based on inventory rules
- **Delivery Tracking**: Monitors delivered quantities
- **Warehouse Assignment**: Links orders to fulfillment warehouses

## Entity Relationships & Integrations

### Partner Integration
- Customer management through Partners plugin
- Separate invoice and shipping addresses
- Contact integration for communications

### Product Integration
- Full product catalog integration
- UOM management and conversions
- Packaging support and configurations
- Product-based pricing with discounts

### Inventory Integration
- Warehouse assignment for fulfillment
- Automatic stock movement generation
- Real-time delivery tracking
- Configurable inventory replenishment rules

### Accounting Integration
- Automatic invoice generation
- Complex tax computation
- Payment terms configuration
- Proper accounting move generation

## API Requirements

### Core CRUD Operations
```http
GET    /api/sales/orders
POST   /api/sales/orders
GET    /api/sales/orders/{id}
PUT    /api/sales/orders/{id}
DELETE /api/sales/orders/{id}
```

### State Management
```http
PUT    /api/sales/orders/{id}/confirm
PUT    /api/sales/orders/{id}/cancel
PUT    /api/sales/orders/{id}/lock
POST   /api/sales/orders/{id}/send-email
```

### Line Management
```http
GET    /api/sales/orders/{id}/lines
POST   /api/sales/orders/{id}/lines
PUT    /api/sales/orders/{id}/lines/{line_id}
DELETE /api/sales/orders/{id}/lines/{line_id}
```

### Delivery Management
```http
GET    /api/sales/orders/{id}/deliveries
POST   /api/sales/orders/{id}/deliveries/create
PUT    /api/sales/orders/{id}/deliveries/{delivery_id}/validate
```

### Invoice Management
```http
GET    /api/sales/orders/{id}/invoices
POST   /api/sales/orders/{id}/invoices/create
POST   /api/sales/orders/{id}/invoices/advance-payment
```

### Team Management
```http
GET    /api/sales/teams
POST   /api/sales/teams
GET    /api/sales/teams/{id}/performance
PUT    /api/sales/teams/{id}/targets
```

## Advanced Features

### Quotation Templates
- Predefined quotation templates with standard products
- Configurable terms and conditions
- Digital signature workflows
- Upfront payment requirements

### Sales Team Management
- Performance tracking and metrics
- Configurable sales targets
- Commission calculation
- Territory management

### Delivery Integration
- Multiple delivery methods (manual/automatic)
- Partial delivery support
- Real-time status tracking
- Customer notifications

### Analytics & Reporting
- Sales pipeline tracking
- Margin analysis reporting
- Performance dashboards
- Sales forecasting

This Sales plugin provides comprehensive sales order management from quotation through delivery and invoicing, with deep integration across the ERP system.