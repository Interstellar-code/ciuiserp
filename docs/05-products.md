# Products Plugin Documentation

## Overview

The Products plugin is a foundational component of the CiuisERP system that manages the complete product catalog, including categories, attributes, packaging, pricing, and supplier information. It serves as the core foundation for both Accounts (taxation) and Inventories (stock management) plugins.

## Plugin Structure

**Location**: `/mnt/c/laragon/www/ciuiserp/plugins/webkul/products/`

**Provider**: `Webkul\Product\ProductServiceProvider`

**Namespace**: `Webkul\Product`

## Database Schema

### Core Product Tables

#### 1. products_categories
Hierarchical product categorization system with circular reference protection.

```sql
- id (Primary Key)
- name (indexed) - Category name
- full_name - Complete hierarchical path name (e.g., "Electronics / Computers / Laptops")
- parent_path - Path of parent IDs (e.g., "/1/5/")
- parent_id (Foreign Key to products_categories) - Self-referencing hierarchy
- creator_id (Foreign Key to users)
- timestamps
```

**Key Features**:
- Hierarchical structure with unlimited depth
- Automatic circular reference validation
- Auto-generated full_name and parent_path fields
- Cascade delete for child categories

#### 2. products_products
Main product entity supporting both goods and services.

```sql
- id (Primary Key)
- type (enum: 'goods', 'service') - Product type
- name - Product name
- service_tracking (default: 'none') - Service tracking method
- reference - Internal product reference/SKU
- barcode - Product barcode
- price (decimal 15,4) - Base selling price
- cost (decimal 15,4) - Cost price
- volume (decimal 15,4) - Product volume
- weight (decimal 15,4) - Product weight
- description (text) - General description
- description_purchase (text) - Purchase-specific description
- description_sale (text) - Sales-specific description
- enable_sales (boolean) - Can be sold
- enable_purchase (boolean) - Can be purchased
- is_favorite (boolean) - Favorite status
- is_configurable (boolean) - Has variants/configurations
- sort (integer) - Display order
- images (json) - Product images array
- parent_id (Foreign Key to products_products) - For product variants
- uom_id (Foreign Key to unit_of_measures) - Unit of measure
- uom_po_id (Foreign Key to unit_of_measures) - Purchase unit of measure
- category_id (Foreign Key to products_categories) - Product category
- company_id (Foreign Key to companies)
- creator_id (Foreign Key to users)
- soft_deletes
- timestamps
```

**Extended Fields** (added by Inventories plugin):
```sql
- sale_delay (integer) - Sales delivery delay
- tracking (enum) - Inventory tracking method
- description_picking (text) - Picking description
- description_pickingout (text) - Outbound picking description
- description_pickingin (text) - Inbound picking description
- is_storable (boolean) - Can be stored in inventory
- expiration_time (integer) - Expiration time in days
- use_time (integer) - Best before time in days
- removal_time (integer) - Removal time in days
- alert_time (integer) - Alert time in days
- use_expiration_date (boolean) - Uses expiration dates
- responsible_id (Foreign Key to users) - Responsible person
```

#### 3. products_tags
Product tagging system for flexible categorization.

```sql
- id (Primary Key)
- name - Tag name
- color - Tag color
- creator_id (Foreign Key to users)
- timestamps
```

#### 4. products_product_tag
Many-to-many relationship between products and tags.

```sql
- product_id (Foreign Key to products_products)
- tag_id (Foreign Key to products_tags)
```

### Product Attributes System

#### 5. products_attributes
Define configurable product attributes (color, size, etc.).

```sql
- id (Primary Key)
- name - Attribute name
- type (enum: 'radio', 'select', 'color') - Input type
- sort (integer) - Display order
- creator_id (Foreign Key to users)
- soft_deletes
- timestamps
```

#### 6. products_attribute_options
Available options for each attribute.

```sql
- id (Primary Key)
- name - Option name
- color - Color code (for color attributes)
- extra_price (decimal 15,4) - Additional price for this option
- sort (integer) - Display order
- attribute_id (Foreign Key to products_attributes)
- creator_id (Foreign Key to users)
- timestamps
```

#### 7. products_product_attributes
Links products to available attributes.

```sql
- id (Primary Key)
- sort (integer) - Display order
- product_id (Foreign Key to products_products)
- attribute_id (Foreign Key to products_attributes)
- creator_id (Foreign Key to users)
- timestamps
```

#### 8. products_product_attribute_values
Stores specific attribute values for products/variants.

```sql
- id (Primary Key)
- extra_price (decimal 15,4) - Price modification for this value
- product_id (Foreign Key to products_products)
- attribute_id (Foreign Key to products_attributes)
- product_attribute_id (Foreign Key to products_product_attributes)
- attribute_option_id (Foreign Key to products_attribute_options)
```

#### 9. products_product_combinations
Links product variants to their attribute combinations.

```sql
- id (Primary Key)
- product_id (Foreign Key to products_products) - Variant product
- product_attribute_value_id (Foreign Key to products_product_attribute_values)
- timestamps
```

### Packaging System

#### 10. products_packagings
Different packaging options for products.

```sql
- id (Primary Key)
- name - Packaging name
- barcode - Packaging-specific barcode
- qty (decimal 12,4) - Quantity in this packaging
- sort (integer) - Display order
- product_id (Foreign Key to products_products)
- creator_id (Foreign Key to users)
- company_id (Foreign Key to companies)
- timestamps
```

### Pricing System

#### 11. products_price_rules
Master price rule definitions.

```sql
- id (Primary Key)
- name - Price rule name
- sort (integer) - Priority order
- currency_id (Foreign Key to currencies)
- company_id (Foreign Key to companies)
- creator_id (Foreign Key to users)
- soft_deletes
- timestamps
```

#### 12. products_price_rule_items
Detailed pricing rules with conditions and calculations.

```sql
- id (Primary Key)
- apply_to (enum) - What to apply rule to
- display_apply_to - Display name for application
- base (enum) - Base price calculation method
- type (enum: 'percentage', 'formula', 'fixed') - Rule type
- min_quantity (decimal 15,4) - Minimum quantity for rule
- fixed_price (decimal 15,4) - Fixed price amount
- price_discount (decimal 15,4) - Discount amount
- price_round (decimal 15,4) - Rounding factor
- price_surcharge (decimal 15,4) - Additional surcharge
- price_markup (decimal 15,4) - Markup amount
- price_min_margin (decimal 15,4) - Minimum margin
- percent_price (decimal 15,4) - Percentage adjustment
- starts_at (datetime) - Rule validity start
- ends_at (datetime) - Rule validity end
- price_rule_id (Foreign Key to products_price_rules)
- base_price_rule_id (Foreign Key to products_price_rules) - Base rule reference
- product_id (Foreign Key to products_products)
- category_id (Foreign Key to products_categories)
- currency_id (Foreign Key to currencies)
- company_id (Foreign Key to companies)
- creator_id (Foreign Key to users)
- timestamps
```

#### 13. products_product_price_lists
Price list management for different markets/customers.

```sql
- id (Primary Key)
- sort (integer) - Display order
- currency_id (Foreign Key to currencies)
- company_id (Foreign Key to companies)
- creator_id (Foreign Key to users)
- name - Price list name
- is_active (boolean, default: true) - Active status
- timestamps
```

### Supplier Management

#### 14. products_product_suppliers
Supplier information and pricing for products.

```sql
- id (Primary Key)
- sort (integer) - Priority order
- delay (integer) - Delivery delay in days
- product_name - Supplier's product name
- product_code - Supplier's product code
- starts_at (date) - Valid from date
- ends_at (date) - Valid to date
- min_qty (decimal) - Minimum order quantity
- price (decimal) - Supplier price
- discount (decimal) - Discount percentage
- product_id (Foreign Key to products_products)
- partner_id (Foreign Key to partners_partners) - Supplier
- currency_id (Foreign Key to currencies)
- company_id (Foreign Key to companies)
- creator_id (Foreign Key to users)
- timestamps
```

## Business Logic Models

### Core Models

#### Product Model
**Location**: `/mnt/c/laragon/www/ciuiserp/plugins/webkul/products/src/Models/Product.php`

**Key Features**:
- Sortable implementation
- Soft deletes
- Activity logging (HasLogActivity trait)
- Chatter integration (HasChatter trait)
- Product type enum casting
- Image JSON casting

**Relationships**:
- `parent()` - Parent product (for variants)
- `variants()` - Child variants
- `category()` - Product category
- `tags()` - Many-to-many with tags
- `attributes()` - Product attributes
- `attribute_values()` - Attribute values
- `combinations()` - Variant combinations
- `uom()` - Unit of measure
- `uomPO()` - Purchase unit of measure
- `priceRuleItems()` - Price rules
- `supplierInformation()` - Supplier details
- `company()` - Company
- `creator()` - Creator user

#### Category Model
**Location**: `/mnt/c/laragon/www/ciuiserp/plugins/webkul/products/src/Models/Category.php`

**Key Features**:
- Hierarchical structure with unlimited depth
- Automatic circular reference prevention
- Auto-generated full_name path
- Activity logging
- Chatter integration

**Relationships**:
- `parent()` - Parent category
- `children()` - Child categories
- `products()` - Products in category
- `priceRuleItems()` - Price rules for category
- `creator()` - Creator user

#### Attribute Model
**Location**: `/mnt/c/laragon/www/ciuiserp/plugins/webkul/products/src/Models/Attribute.php`

**Key Features**:
- Sortable implementation
- Soft deletes
- Attribute type enum casting

**Relationships**:
- `options()` - Attribute options
- `creator()` - Creator user

## Enums

### ProductType
**Values**:
- `GOODS` - Physical products
- `SERVICE` - Service products

### AttributeType
**Values**:
- `RADIO` - Radio button selection
- `SELECT` - Dropdown selection
- `COLOR` - Color picker

### PriceRuleType
**Values**:
- `PERCENTAGE` - Percentage-based pricing
- `FORMULA` - Formula-based calculation
- `FIXED` - Fixed price

### Additional Enums
- `PriceRuleApplyTo` - What the rule applies to
- `PriceRuleBase` - Base calculation method
- `ProductRemoval` - Product removal strategies

## Entity Relationships

### Integration with Accounts Plugin

The Products plugin integrates with the Accounts plugin through:

1. **Product Taxes** (`accounts_product_taxes`):
   - Links products to applicable taxes
   - Cascade deletion when product is removed

2. **Supplier Taxes** (`accounts_product_supplier_taxes`):
   - Links product suppliers to applicable taxes
   - Used for purchase tax calculations

3. **Move Lines** (`accounts_account_move_lines`):
   - References products in accounting entries
   - Tracks product-related financial transactions

### Integration with Inventories Plugin

The Products plugin is extended by the Inventories plugin:

1. **Product Table Extensions**:
   - Inventory tracking fields
   - Expiration and lot management
   - Storage and handling descriptions

2. **Product Quantities** (`inventories_product_quantities`):
   - Stock levels by location and lot
   - Reserved quantities
   - Inventory adjustments

3. **Inventory Operations**:
   - Product movements
   - Lot tracking
   - Stock valuation

### Integration with Sales/Purchases

Products serve as the foundation for:
- Sales order lines
- Purchase order lines
- Invoice lines
- Quotation items

## API Requirements

### Product Catalog APIs

#### Product Management
```php
// List products with filtering
GET /api/products
- Filters: category, type, tags, company
- Search: name, reference, barcode
- Sorting: name, price, created_at

// Create product
POST /api/products
{
    "name": "Product Name",
    "type": "goods",
    "category_id": 1,
    "price": 100.00,
    "cost": 80.00,
    "description": "Product description",
    "enable_sales": true,
    "enable_purchase": true
}

// Update product
PUT /api/products/{id}

// Get product details
GET /api/products/{id}
- Includes: category, tags, attributes, variants, suppliers

// Delete product
DELETE /api/products/{id}
```

#### Category Management
```php
// List categories (hierarchical)
GET /api/categories
- Returns: tree structure with full_name paths

// Create category
POST /api/categories
{
    "name": "Category Name",
    "parent_id": 1
}

// Update category
PUT /api/categories/{id}

// Get category with products
GET /api/categories/{id}/products
```

#### Attribute Handling
```php
// List attributes
GET /api/attributes

// Create attribute with options
POST /api/attributes
{
    "name": "Color",
    "type": "color",
    "options": [
        {"name": "Red", "color": "#FF0000"},
        {"name": "Blue", "color": "#0000FF"}
    ]
}

// Assign attributes to product
POST /api/products/{id}/attributes
{
    "attribute_id": 1,
    "values": [
        {"option_id": 1, "extra_price": 0},
        {"option_id": 2, "extra_price": 5.00}
    ]
}

// Generate product variants
POST /api/products/{id}/generate-variants
```

#### Pricing APIs
```php
// Create price rule
POST /api/price-rules
{
    "name": "Volume Discount",
    "currency_id": 1,
    "items": [
        {
            "apply_to": "product",
            "product_id": 1,
            "type": "percentage",
            "min_quantity": 10,
            "price_discount": 10.0
        }
    ]
}

// Get product pricing
GET /api/products/{id}/pricing
- Parameters: quantity, customer_id, date

// Price list management
GET /api/price-lists
POST /api/price-lists
PUT /api/price-lists/{id}
```

#### Packaging APIs
```php
// List product packagings
GET /api/products/{id}/packagings

// Create packaging
POST /api/products/{id}/packagings
{
    "name": "Box of 12",
    "qty": 12,
    "barcode": "1234567890123"
}
```

### Advanced Features

#### Product Variants
- Support for configurable products with multiple attributes
- Automatic variant generation based on attribute combinations
- Individual pricing and inventory tracking per variant

#### Supplier Management
- Multiple suppliers per product
- Time-based supplier pricing
- Minimum order quantities
- Delivery lead times

#### Pricing Engine
- Rule-based pricing with conditions
- Volume discounts
- Time-based pricing
- Currency-specific price lists
- Margin calculations

#### Search and Filtering
- Full-text search across name, description, reference
- Category-based filtering
- Attribute-based filtering
- Tag-based organization
- Company-specific product catalogs

## Data Validation Rules

### Product Validation
- Name: Required, max 255 characters
- Type: Required, must be 'goods' or 'service'
- Category: Required, must exist
- Price: Required, minimum 0, max 15 digits with 4 decimals
- Reference: Optional, max 255 characters, unique per company
- Barcode: Optional, max 255 characters

### Category Validation
- Name: Required, max 255 characters
- Parent: Optional, must exist, cannot create circular references
- Automatic validation prevents circular dependencies

### Attribute Validation
- Name: Required, max 255 characters
- Type: Required, must be valid AttributeType
- Options: Required for select/radio types

## Security Considerations

### Access Control
- Product creation/editing requires appropriate permissions
- Company-based data isolation
- User-based creator tracking

### Data Integrity
- Foreign key constraints maintain referential integrity
- Soft deletes prevent accidental data loss
- Circular reference prevention in categories
- Cascade deletes where appropriate

## Performance Optimizations

### Database Indexing
- Name field indexed for fast searching
- Foreign keys indexed automatically
- Category parent_path for efficient hierarchy queries

### Caching Strategy
- Cache category hierarchies
- Cache frequently accessed product data
- Cache price calculations

### Query Optimization
- Eager loading of relationships
- Efficient category tree traversal
- Optimized product search queries

## Future Enhancements

### Planned Features
1. **Product Reviews and Ratings**
2. **Product Bundles and Kits**
3. **Product Lifecycle Management**
4. **Advanced Pricing Rules**
5. **Product Comparison Tools**
6. **Multi-language Product Descriptions**
7. **Product Import/Export Tools**
8. **Advanced Search with Elasticsearch**

### API Versioning
- Current API version: v1
- Backward compatibility maintained
- Deprecation notices for removed features

## Troubleshooting

### Common Issues
1. **Circular Category References**: Automatic validation prevents this
2. **Missing UOM**: Default UOM must exist before creating products
3. **Price Calculation Errors**: Check price rule order and validity dates
4. **Variant Generation Failures**: Ensure all required attributes have options

### Debugging Tools
- Activity logs track all changes
- Soft deletes allow data recovery
- Comprehensive error messages
- Debug mode reveals SQL queries

## Conclusion

The Products plugin serves as the foundational component for product catalog management in CiuisERP. Its comprehensive feature set supports complex product hierarchies, configurable attributes, sophisticated pricing rules, and seamless integration with accounting and inventory systems. The robust API design enables both internal system integration and external application connectivity.