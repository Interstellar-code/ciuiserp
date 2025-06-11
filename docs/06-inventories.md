# Inventories Plugin Documentation

## Overview

The Inventories plugin is a sophisticated warehouse and stock management system for the CiuiSERP platform. It provides comprehensive inventory operations management, multi-location warehouse management, automated workflow routing, lot tracking, and real-time inventory tracking capabilities.

## Table of Contents

1. [Database Schema](#database-schema)
2. [Core Models](#core-models)
3. [Business Logic](#business-logic)
4. [Entity Relationships](#entity-relationships)
5. [API Requirements](#api-requirements)
6. [Configuration](#configuration)
7. [Workflow Management](#workflow-management)
8. [Integration Points](#integration-points)

---

## Database Schema

### Core Tables

#### Warehouses (`inventories_warehouses`)
Main warehouse entities with location hierarchy and configuration.

```sql
CREATE TABLE inventories_warehouses (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(255),
    sort INTEGER,
    reception_steps VARCHAR(255),    -- Enum: ReceptionStep
    delivery_steps VARCHAR(255),     -- Enum: DeliveryStep
    company_id BIGINT,
    partner_address_id BIGINT,
    creator_id BIGINT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    deleted_at TIMESTAMP,
    
    UNIQUE(company_id, name),
    UNIQUE(company_id, code)
);
```

#### Locations (`inventories_locations`)
Hierarchical storage locations within warehouses with 3D positioning.

```sql
CREATE TABLE inventories_locations (
    id BIGINT PRIMARY KEY,
    position_x INTEGER DEFAULT 0,    -- Corridor X
    position_y INTEGER DEFAULT 0,    -- Shelves Y  
    position_z INTEGER DEFAULT 0,    -- Height Z
    type VARCHAR(255),               -- Enum: LocationType
    name VARCHAR(255),
    full_name VARCHAR(255),
    description VARCHAR(255),
    parent_path VARCHAR(255),
    barcode VARCHAR(255),
    removal_strategy VARCHAR(255),
    cyclic_inventory_frequency INTEGER DEFAULT 0,
    last_inventory_date DATE,
    next_inventory_date DATE,
    is_scrap BOOLEAN DEFAULT 0,
    is_replenish BOOLEAN DEFAULT 0,
    is_dock BOOLEAN DEFAULT 0,
    parent_id BIGINT,
    company_id BIGINT,
    storage_category_id BIGINT,
    warehouse_id BIGINT,
    creator_id BIGINT,
    
    UNIQUE(company_id, barcode)
);
```

#### Operations (`inventories_operations`)
Main inventory operations (receipts, deliveries, internal transfers).

```sql
CREATE TABLE inventories_operations (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    origin VARCHAR(255),
    move_type VARCHAR(255) DEFAULT 'direct',  -- Enum: MoveType
    state VARCHAR(255),                       -- Enum: OperationState
    is_favorite BOOLEAN DEFAULT 0,
    has_deadline_issue BOOLEAN DEFAULT 0,
    is_printed BOOLEAN DEFAULT 0,
    is_locked BOOLEAN DEFAULT 0,
    deadline DATETIME,
    scheduled_at DATETIME,
    closed_at DATETIME,
    user_id BIGINT,
    owner_id BIGINT,
    operation_type_id BIGINT,
    source_location_id BIGINT,
    destination_location_id BIGINT,
    back_order_id BIGINT,
    return_id BIGINT,
    partner_id BIGINT,
    company_id BIGINT,
    creator_id BIGINT,
    sale_order_id BIGINT
);
```

#### Moves (`inventories_moves`)
Individual product movements within operations.

```sql
CREATE TABLE inventories_moves (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    state VARCHAR(255),                -- Enum: MoveState
    origin VARCHAR(255),
    procure_method VARCHAR(255) DEFAULT 'make_to_stock',
    reference VARCHAR(255),
    description_picking TEXT,
    next_serial VARCHAR(255),
    next_serial_count INTEGER,
    is_favorite BOOLEAN DEFAULT 0,
    product_qty DECIMAL(15,4) DEFAULT 0,
    product_uom_qty DECIMAL(15,4) DEFAULT 0,
    quantity DECIMAL(15,4) DEFAULT 0,
    is_picked BOOLEAN DEFAULT 0,
    is_scraped BOOLEAN DEFAULT 0,
    is_inventory BOOLEAN DEFAULT 0,
    is_refund BOOLEAN DEFAULT 0,
    reservation_date DATE,
    scheduled_at DATETIME,
    deadline DATETIME,
    alert_date DATETIME,
    operation_id BIGINT,
    product_id BIGINT,
    uom_id BIGINT,
    source_location_id BIGINT,
    destination_location_id BIGINT,
    final_location_id BIGINT,
    partner_id BIGINT,
    scrap_id BIGINT,
    rule_id BIGINT,
    operation_type_id BIGINT,
    origin_returned_move_id BIGINT,
    restrict_partner_id BIGINT,
    warehouse_id BIGINT,
    package_level_id BIGINT,
    product_packaging_id BIGINT,
    company_id BIGINT,
    creator_id BIGINT
);
```

#### Move Lines (`inventories_move_lines`)
Detailed line items for moves with lot and package tracking.

```sql
CREATE TABLE inventories_move_lines (
    id BIGINT PRIMARY KEY,
    lot_name VARCHAR(255),
    state VARCHAR(255),
    reference VARCHAR(255),
    picking_description VARCHAR(255),
    qty DECIMAL(15,4) DEFAULT 0,
    uom_qty DECIMAL(15,4) DEFAULT 0,
    is_picked BOOLEAN DEFAULT 0,
    scheduled_at DATETIME,
    move_id BIGINT,
    operation_id BIGINT,
    product_id BIGINT,
    uom_id BIGINT,
    package_id BIGINT,
    result_package_id BIGINT,
    package_level_id BIGINT,
    lot_id BIGINT,
    partner_id BIGINT,
    source_location_id BIGINT,
    destination_location_id BIGINT,
    company_id BIGINT,
    creator_id BIGINT
);
```

#### Product Quantities (`inventories_product_quantities`)
Real-time inventory quantities by location, lot, and package.

```sql
CREATE TABLE inventories_product_quantities (
    id BIGINT PRIMARY KEY,
    quantity DECIMAL(15,4) DEFAULT 0,
    reserved_quantity DECIMAL(15,4) DEFAULT 0,
    counted_quantity DECIMAL(15,4) DEFAULT 0,
    difference_quantity DECIMAL(15,4) DEFAULT 0,
    inventory_diff_quantity DECIMAL(15,4) DEFAULT 0,
    inventory_quantity_set BOOLEAN DEFAULT 0,
    scheduled_at DATE,
    incoming_at DATETIME,
    product_id BIGINT,
    location_id BIGINT,
    storage_category_id BIGINT,
    lot_id BIGINT,
    package_id BIGINT,
    partner_id BIGINT,
    user_id BIGINT,
    company_id BIGINT,
    creator_id BIGINT
);
```

### Configuration Tables

#### Operation Types (`inventories_operation_types`)
```sql
CREATE TABLE inventories_operation_types (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    type VARCHAR(255),               -- Enum: OperationType
    sequence_code VARCHAR(255),
    reservation_method VARCHAR(255), -- Enum: ReservationMethod
    reservation_days_before INTEGER,
    reservation_days_before_priority INTEGER,
    product_label_format VARCHAR(255),
    lot_label_format VARCHAR(255),
    package_label_to_print VARCHAR(255),
    barcode VARCHAR(255),
    create_backorder VARCHAR(255),   -- Enum: CreateBackorder
    move_type VARCHAR(255),          -- Enum: MoveType
    show_entire_packs BOOLEAN DEFAULT 0,
    use_create_lots BOOLEAN DEFAULT 0,
    use_existing_lots BOOLEAN DEFAULT 0,
    print_label BOOLEAN DEFAULT 0,
    show_operations BOOLEAN DEFAULT 0,
    auto_show_reception_report BOOLEAN DEFAULT 0,
    auto_print_delivery_slip BOOLEAN DEFAULT 0,
    auto_print_return_slip BOOLEAN DEFAULT 0,
    auto_print_product_labels BOOLEAN DEFAULT 0,
    auto_print_lot_labels BOOLEAN DEFAULT 0,
    auto_print_reception_report BOOLEAN DEFAULT 0,
    auto_print_reception_report_labels BOOLEAN DEFAULT 0,
    auto_print_packages BOOLEAN DEFAULT 0,
    auto_print_package_label BOOLEAN DEFAULT 0,
    return_operation_type_id BIGINT,
    source_location_id BIGINT,
    destination_location_id BIGINT,
    warehouse_id BIGINT,
    company_id BIGINT,
    creator_id BIGINT
);
```

#### Routes and Rules (`inventories_routes`, `inventories_rules`)
Automated workflow routing configuration.

```sql
CREATE TABLE inventories_routes (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    active BOOLEAN DEFAULT 1,
    sort INTEGER,
    company_id BIGINT,
    creator_id BIGINT
);

CREATE TABLE inventories_rules (
    id BIGINT PRIMARY KEY,
    sort INTEGER,
    name VARCHAR(255),
    route_sort INTEGER,
    delay INTEGER,
    group_propagation_option VARCHAR(255), -- Enum: GroupPropagation
    action VARCHAR(255),                   -- Enum: RuleAction
    procure_method VARCHAR(255),           -- Enum: ProcureMethod
    auto VARCHAR(255),                     -- Enum: RuleAuto
    push_domain VARCHAR(255),
    location_dest_from_rule BOOLEAN DEFAULT 0,
    propagate_cancel BOOLEAN DEFAULT 0,
    propagate_carrier BOOLEAN DEFAULT 0,
    source_location_id BIGINT,
    destination_location_id BIGINT,
    route_id BIGINT,
    operation_type_id BIGINT,
    partner_address_id BIGINT,
    warehouse_id BIGINT,
    propagate_warehouse_id BIGINT,
    company_id BIGINT,
    creator_id BIGINT
);
```

#### Lots and Packages
```sql
CREATE TABLE inventories_lots (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    reference VARCHAR(255),
    expiration_date DATE,
    use_expiration_date BOOLEAN DEFAULT 0,
    removal_date DATE,
    alert_date DATE,
    company_id BIGINT,
    product_id BIGINT,
    creator_id BIGINT
);

CREATE TABLE inventories_packages (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    pack_date DATETIME,
    company_id BIGINT,
    package_type_id BIGINT,
    location_id BIGINT,
    creator_id BIGINT
);
```

---

## Core Models

### Warehouse Model
- **File**: `/plugins/webkul/inventories/src/Models/Warehouse.php`
- **Key Features**:
  - Hierarchical location management
  - Multi-step reception and delivery configuration
  - Integration with operation types and routes
  - Warehouse-to-warehouse resupply relationships

### Operation Model
- **File**: `/plugins/webkul/inventories/src/Models/Operation.php`
- **Key Features**:
  - State management (Draft → Confirmed → Assigned → Done)
  - Back order and return handling
  - Integration with purchase and sale orders
  - Automatic name generation and propagation

### Move Model
- **File**: `/plugins/webkul/inventories/src/Models/Move.php`
- **Key Features**:
  - Product quantity tracking with UOM conversion
  - Reservation and allocation logic
  - Push rule integration
  - Serial number and lot tracking support

### ProductQuantity Model
- **File**: `/plugins/webkul/inventories/src/Models/ProductQuantity.php`
- **Key Features**:
  - Real-time inventory tracking
  - Reserved quantity management
  - Automatic inventory scheduling
  - Multi-dimensional tracking (location, lot, package)

### Rule Model
- **File**: `/plugins/webkul/inventories/src/Models/Rule.php`
- **Key Features**:
  - Automated workflow routing
  - Push/Pull rule actions
  - Delay and propagation settings
  - Priority-based rule execution

---

## Business Logic

### Inventory Manager
**File**: `/plugins/webkul/inventories/src/InventoryManager.php`

#### Core Methods

##### Transfer Validation Flow
```php
public function validateTransfer(Operation $record): Operation
```
- Validates and processes inventory transfers
- Updates product quantities in source and destination locations
- Handles lot tracking and package movements
- Triggers push rules for automated workflows
- Integrates with purchase and sale order updates

##### Transfer Computation
```php
public function computeTransfer(Operation $record): Operation
```
- Calculates available quantities for moves
- Handles reservation logic and allocation
- Manages move line creation and updates
- Processes backorder scenarios

##### Push Rule System
```php
public function applyPushRules(Operation $record): void
```
- Automatically creates downstream operations
- Routes products through defined workflows
- Handles cross-docking and multi-step processes
- Manages warehouse-to-warehouse transfers

### State Management

#### Operation States
- **DRAFT**: Initial state, can be modified
- **CONFIRMED**: Confirmed but not allocated
- **ASSIGNED**: Stock allocated and reserved
- **DONE**: Transfer completed
- **CANCELED**: Operation canceled

#### Move States
- **DRAFT**: Initial move state
- **CONFIRMED**: Confirmed but no stock available
- **ASSIGNED**: Stock fully allocated
- **PARTIALLY_ASSIGNED**: Stock partially allocated
- **DONE**: Move completed
- **CANCELED**: Move canceled

### Automation Features

#### Push Rules
Automated workflows that create follow-up operations:
- **Cross-docking**: Direct customer to supplier routing
- **Quality Control**: Automatic QC step insertion
- **Multi-step Delivery**: Pack → Ship workflows
- **Drop-shipping**: Direct supplier to customer

#### Procurement Methods
- **Make to Stock**: Standard inventory management
- **Make to Order**: Customer-specific production
- **Drop Ship**: Direct supplier fulfillment

---

## Entity Relationships

### Core Relationships

```
Company
├── Warehouses
│   ├── Locations (hierarchical)
│   ├── Operation Types
│   └── Routes
├── Operations
│   ├── Moves
│   │   └── Move Lines
│   └── Integration with Sales/Purchase Orders
└── Product Quantities
    ├── By Location
    ├── By Lot
    └── By Package
```

### Integration Points

#### Products Module (Required)
- Product master data and attributes
- Unit of measure conversions
- Product categories and routing
- Packaging definitions

#### Partners Module
- Supplier and customer information
- Address management for warehouses
- Partner-specific routing rules

#### Purchase Module Integration
- Purchase order to receipt conversion
- Vendor drop-ship operations
- Purchase order operation tracking

#### Sales Module Integration
- Sales order to delivery conversion
- Customer-specific delivery requirements
- Backorder management

#### Accounts Module Integration
- Inventory valuation impacts
- Cost accounting for movements
- Automated journal entries

---

## API Requirements

### Warehouse Management APIs

#### Create/Update Warehouse
```http
POST /api/warehouses
PUT /api/warehouses/{id}

{
    "name": "Main Warehouse",
    "code": "WH001",
    "reception_steps": "one_step",
    "delivery_steps": "ship_only",
    "partner_address_id": 1,
    "company_id": 1
}
```

#### Get Warehouse Inventory
```http
GET /api/warehouses/{id}/inventory

Response:
{
    "warehouse": {...},
    "inventory": [
        {
            "product_id": 1,
            "location_id": 5,
            "quantity": 100.0,
            "reserved_quantity": 20.0,
            "available_quantity": 80.0,
            "lot_id": 3,
            "package_id": 7
        }
    ]
}
```

### Operation Management APIs

#### Create Operation
```http
POST /api/operations

{
    "operation_type_id": 1,
    "source_location_id": 5,
    "destination_location_id": 10,
    "partner_id": 2,
    "scheduled_at": "2025-02-01T10:00:00Z",
    "moves": [
        {
            "product_id": 1,
            "product_qty": 50.0,
            "uom_id": 1
        }
    ]
}
```

#### Process Operation Actions
```http
POST /api/operations/{id}/validate
POST /api/operations/{id}/cancel
POST /api/operations/{id}/return
```

### Real-time Inventory APIs

#### Get Product Quantities
```http
GET /api/products/{id}/quantities

Response:
{
    "product_id": 1,
    "total_quantity": 500.0,
    "available_quantity": 450.0,
    "reserved_quantity": 50.0,
    "locations": [
        {
            "location_id": 5,
            "location_name": "Stock/WH001",
            "quantity": 300.0,
            "available": 270.0,
            "reserved": 30.0
        }
    ]
}
```

#### Stock Movement History
```http
GET /api/products/{id}/movements

Response:
{
    "movements": [
        {
            "move_id": 100,
            "operation_name": "WH001/OUT/00001",
            "quantity": -10.0,
            "source_location": "Stock/WH001",
            "destination_location": "Customers",
            "date": "2025-01-15T14:30:00Z"
        }
    ]
}
```

### Automated Workflow APIs

#### Apply Push Rules
```http
POST /api/operations/{id}/apply-push-rules

Response:
{
    "created_operations": [
        {
            "operation_id": 201,
            "name": "WH001/PACK/00015",
            "rule_applied": "Auto Pack Rule"
        }
    ]
}
```

#### Replenishment Suggestions
```http
GET /api/warehouses/{id}/replenishment

Response:
{
    "suggestions": [
        {
            "product_id": 1,
            "current_quantity": 5.0,
            "minimum_quantity": 20.0,
            "suggested_quantity": 100.0,
            "supplier_id": 3
        }
    ]
}
```

---

## Configuration

### Settings Groups

#### Warehouse Settings
**File**: `/plugins/webkul/inventories/src/Settings/WarehouseSettings.php`

```php
class WarehouseSettings extends Settings
{
    public bool $enable_locations;           // Enable location management
    public bool $enable_multi_steps_routes;  // Enable complex workflows
}
```

#### Traceability Settings
**File**: `/plugins/webkul/inventories/src/Settings/TraceabilitySettings.php`

```php
class TraceabilitySettings extends Settings
{
    public bool $enable_lots_serial_numbers;     // Enable lot tracking
    public bool $enable_expiration_dates;        // Enable expiry management
    public bool $display_on_delivery_slips;      // Show lots on documents
    public bool $enable_consignments;            // Enable consignment stock
}
```

#### Operation Settings
Global operation configuration including reservation policies and automation rules.

### Enums and Constants

#### Location Types
- **SUPPLIER**: Supplier locations
- **STOCK**: Internal stock locations
- **CUSTOMER**: Customer locations
- **INVENTORY**: Inventory adjustment locations
- **PRODUCTION**: Manufacturing locations
- **TRANSIT**: Transit locations

#### Operation Types
- **RECEIPT**: Incoming goods
- **DELIVERY**: Outgoing goods
- **INTERNAL**: Internal transfers
- **MANUFACTURING**: Production operations
- **INVENTORY**: Stock adjustments

---

## Workflow Management

### Reception Workflows
1. **One Step**: Direct to stock
2. **Two Steps**: Receive → Quality Control → Stock
3. **Three Steps**: Receive → Quality Control → Store

### Delivery Workflows
1. **Ship Only**: Direct from stock to customer
2. **Pick + Ship**: Pick → Pack → Ship
3. **Pick + Pack + Ship**: Pick → Pack → Ship with quality checks

### Push Rule Execution
Automated workflow routing based on:
- Product routing rules
- Location-based rules
- Warehouse-specific workflows
- Partner preferences

### Quality Control Integration
- Automatic QC step insertion
- Inspection location routing
- Accept/reject workflows
- Batch testing capabilities

---

## Integration Points

### Cross-Module Integration

#### With Products Module
- Product master data synchronization
- Unit of measure handling
- Product routing configuration
- Packaging and variant support

#### With Purchase Module
- Purchase order to receipt conversion
- Vendor management integration
- Cost tracking and valuation
- Drop-ship order processing

#### With Sales Module
- Sales order to delivery conversion
- Customer delivery preferences
- Backorder processing
- Return merchandise authorization

#### With Accounts Module
- Inventory valuation updates
- Cost of goods sold calculation
- Automated journal entries
- Multi-currency support

### External System Integration

#### Barcode and RFID Support
- Location barcode scanning
- Product identification
- Package tracking
- Lot number validation

#### Warehouse Management Systems (WMS)
- Pick list optimization
- Route planning
- Slotting optimization
- Cycle counting integration

#### Transportation Management
- Shipping integration
- Carrier selection
- Tracking number generation
- Delivery confirmation

---

## Performance Considerations

### Database Optimization
- Indexes on frequently queried columns
- Partitioning for large quantity tables
- Archive strategies for historical data
- Query optimization for real-time inventory

### Caching Strategy
- Product quantity caching
- Location hierarchy caching
- Rule evaluation caching
- Session-based calculation caching

### Scalability Features
- Horizontal scaling support
- Database sharding capabilities
- Async processing for bulk operations
- Event-driven architecture support

---

## Security and Compliance

### Access Control
- Role-based warehouse access
- Location-specific permissions
- Operation type restrictions
- Company data isolation

### Audit Trail
- Complete movement history
- User action logging
- State change tracking
- Document generation logs

### Compliance Features
- Lot tracking for regulations
- Expiry date management
- Serial number tracking
- Quality assurance workflows

This documentation provides a comprehensive overview of the Inventories plugin's capabilities, database structure, business logic, and integration requirements for the CiuiSERP platform.