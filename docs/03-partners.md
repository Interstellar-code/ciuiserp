# Partners Plugin - CRM Foundation Documentation

## Overview

The Partners plugin serves as the foundational CRM (Customer Relationship Management) system for the CIUIS ERP application. It provides comprehensive partner management capabilities that other plugins depend on for customer/vendor relationships, contact management, and business relationship tracking.

## Architecture Overview

The Partners plugin is designed as a core dependency that provides the fundamental building blocks for managing business relationships across the entire ERP system. It uses a flexible account-based architecture that supports various partner types and hierarchical relationships.

### Key Design Principles

1. **Unified Partner Model**: Uses a single table structure (`partners_partners`) to manage all types of partners
2. **Hierarchical Relationships**: Supports parent-child relationships for complex organizational structures
3. **Flexible Account Types**: Individual, Company, and Address types with specialized behaviors
4. **Integration Foundation**: Provides base models and relationships that other plugins extend
5. **Multi-tenancy Support**: Built-in company association for multi-tenant scenarios

## Database Schema

### Core Tables

#### `partners_partners` - Main Partner Entity
```sql
id                                  -- Primary key
account_type                        -- ENUM: individual, company, address
sub_type                           -- Partner subtype (default: 'partner')
name                               -- Partner name (indexed)
avatar                             -- Avatar/profile image path
email                              -- Email address
job_title                          -- Job position/title
website                            -- Website URL
tax_id                            -- Tax identification number (indexed)
phone                             -- Phone number (indexed)
mobile                            -- Mobile number (indexed)
color                             -- UI color designation
company_registry                   -- Company registry number (indexed)
reference                         -- Internal reference code (indexed)

-- Address fields (added in v2025.03.28)
street1                           -- Primary street address
street2                           -- Secondary street address
city                              -- City
zip                               -- Postal/ZIP code
state_id                          -- Foreign key to states table
country_id                        -- Foreign key to countries table

-- Relationship fields
parent_id                         -- Self-referencing for hierarchy
creator_id                        -- User who created the record
user_id                          -- Associated system user
title_id                         -- Reference to titles table
company_id                       -- Company association
industry_id                      -- Industry classification

-- Extended accounting fields (added by accounts plugin)
message_bounce                    -- Message bounce count
supplier_rank                     -- Supplier ranking
customer_rank                     -- Customer ranking
invoice_warning                   -- Invoice warning settings
autopost_bills                    -- Auto-posting configuration
credit_limit                      -- Credit limit amount
ignore_abnormal_invoice_date      -- Invoice date validation setting
ignore_abnormal_invoice_amount    -- Invoice amount validation setting
invoice_sending_method            -- Invoice delivery method
invoice_edi_format_store          -- EDI format configuration
trust                            -- Trust level rating
invoice_warn_msg                  -- Invoice warning message ID
debit_limit                       -- Debit limit amount
peppol_endpoint                   -- PEPPOL endpoint for B2B
peppol_eas                        -- PEPPOL EAS identifier
sale_warn                         -- Sales warning settings
sale_warn_msg                     -- Sales warning message
comment                          -- General comments

-- Accounting relationships (added by accounts plugin)
property_account_payable_id       -- Default payable account
property_account_receivable_id    -- Default receivable account
property_account_position_id      -- Account position
property_payment_term_id          -- Payment terms
property_supplier_payment_term_id -- Supplier payment terms
property_inbound_payment_method_line_id  -- Inbound payment method
property_outbound_payment_method_line_id -- Outbound payment method

-- Audit fields
created_at, updated_at, deleted_at
```

#### `partners_bank_accounts` - Partner Banking Information
```sql
id                               -- Primary key
account_number                   -- Bank account number (unique)
account_holder_name              -- Account holder name (auto-filled from partner)
is_active                        -- Account status
can_send_money                   -- Payment capability flag
creator_id                       -- User who created the record
partner_id                       -- Foreign key to partners_partners
bank_id                         -- Foreign key to banks table
created_at, updated_at, deleted_at
```

#### `partners_industries` - Industry Classifications
```sql
id                               -- Primary key
name                            -- Industry name
description                     -- Industry description
is_active                       -- Active status
creator_id                      -- User who created the record
created_at, updated_at, deleted_at
```

#### `partners_titles` - Professional Titles
```sql
id                               -- Primary key
name                            -- Full title name (e.g., "Doctor")
short_name                      -- Abbreviated form (e.g., "Dr.")
creator_id                      -- User who created the record
created_at, updated_at
```

#### `partners_tags` - Partner Categorization Tags
```sql
id                               -- Primary key
name                            -- Tag name (unique)
color                           -- Display color
creator_id                      -- User who created the record
created_at, updated_at, deleted_at
```

#### `partners_partner_tag` - Many-to-Many Relationship
```sql
tag_id                          -- Foreign key to partners_tags
partner_id                      -- Foreign key to partners_partners
```

## Business Logic & Models

### Partner Model (`Webkul\Partner\Models\Partner`)

The Partner model is the central entity that implements several key patterns:

#### Key Features
- **Multiple Authentication Support**: Implements `FilamentUser` for panel access
- **Soft Deletes**: Full audit trail with restore capability
- **Activity Logging**: Integrated with chatter system for communication tracking
- **File Management**: Avatar handling with proper URL generation
- **Hierarchical Structure**: Self-referencing relationships for organizational hierarchy

#### Account Types (Enum)
```php
enum AccountType: string {
    INDIVIDUAL = 'individual'  // Individual person
    COMPANY = 'company'       // Business entity
    ADDRESS = 'address'       // Address record
}
```

#### Core Relationships
```php
// Geographic
country(): BelongsTo          // Geographic location
state(): BelongsTo           // State/province

// Hierarchical
parent(): BelongsTo          // Parent partner (for contacts/addresses)
addresses(): HasMany         // Child addresses
contacts(): HasMany          // Child contacts

// System
creator(): BelongsTo         // User who created the record
user(): BelongsTo           // Associated system user
company(): BelongsTo        // Company association

// Classification
title(): BelongsTo          // Professional title
industry(): BelongsTo       // Industry classification
tags(): BelongsToMany       // Categorization tags

// Financial
bankAccounts(): HasMany     // Banking information
```

#### Address Management
The partner model uses a clever approach for address management:
- Addresses are stored as partner records with `account_type = 'address'`
- They reference the main partner via `parent_id`
- This allows unlimited addresses per partner with full relationship tracking

#### Contact Management
Similar to addresses, contacts are managed as:
- Partner records with `account_type != 'address'`
- Reference main partner via `parent_id`
- Supports nested organizational structures

### BankAccount Model (`Webkul\Partner\Models\BankAccount`)

#### Auto-Fill Behavior
```php
static::creating(function ($bankAccount) {
    $bankAccount->account_holder_name = $bankAccount->partner->name;
});
```

#### Key Features
- **Account Validation**: Unique account numbers across system
- **Money Transfer Control**: `can_send_money` flag for payment restrictions
- **Bank Integration**: References central banks table
- **Partner Synchronization**: Account holder name auto-syncs with partner name

### Industry Model (`Webkul\Partner\Models\Industry`)

Provides standardized industry classifications based on international standards:

#### Pre-seeded Industries
- Administrative/Utilities
- Agriculture
- Construction
- Education
- Energy Supply
- Entertainment
- Finance/Insurance
- Food/Hospitality
- Health/Social
- IT/Communication
- Manufacturing
- Mining
- Real Estate
- Scientific/Technical
- Transportation/Logistics
- And more...

### Tag System (`Webkul\Partner\Models\Tag`)

Flexible categorization system allowing:
- Color-coded visual organization
- Multi-tag assignment per partner
- Custom categorization schemes
- Filtering and grouping capabilities

## Entity Relationships & Dependencies

### How Other Plugins Integrate

#### Sales Plugin Integration
```php
// Sales orders reference partners as customers
class Order extends Model {
    public function partner(): BelongsTo {
        return $this->belongsTo(Partner::class);
    }
}
```

#### Purchases Plugin Integration  
```php
// Purchase orders reference partners as vendors
class PurchaseOrder extends Model {
    public function vendor(): BelongsTo {
        return $this->belongsTo(Partner::class, 'partner_id');
    }
}
```

#### Employees Plugin Integration
```php
// Employees can be linked to partner records
class Employee extends Model {
    public function partner(): BelongsTo {
        return $this->belongsTo(Partner::class);
    }
}
```

#### Accounts Plugin Integration
The accounts plugin heavily extends the partners table with financial fields:
- Payment terms and methods
- Account mappings (receivable/payable)
- Credit limits and trust levels
- Invoice handling preferences
- PEPPOL B2B integration settings

#### Projects Plugin Integration
```php
// Projects can be associated with customer partners
class Project extends Model {
    public function customer(): BelongsTo {
        return $this->belongsTo(Partner::class, 'partner_id');
    }
}
```

## Address Management System

### AddressType Enum
```php
enum AddressType: string {
    PERMANENT = 'permanent'   // Permanent address
    PRESENT = 'present'      // Current address
    INVOICE = 'invoice'      // Billing address
    DELIVERY = 'delivery'    // Shipping address
    OTHER = 'other'          // Custom address type
}
```

### Address Hierarchy Pattern
```php
// Main partner
$company = Partner::create([
    'account_type' => AccountType::COMPANY,
    'name' => 'Acme Corp',
    // ... other fields
]);

// Company address
$address = Partner::create([
    'account_type' => AccountType::ADDRESS,
    'parent_id' => $company->id,
    'name' => 'Headquarters',
    'street1' => '123 Business St',
    'city' => 'Business City',
    // ... address fields
]);

// Contact person
$contact = Partner::create([
    'account_type' => AccountType::INDIVIDUAL,
    'parent_id' => $company->id,
    'name' => 'John Doe',
    'job_title' => 'Sales Manager',
    // ... contact fields
]);
```

## API Requirements for CRM Operations

### Core CRM API Endpoints

#### Partner Management
```php
// List partners with filtering
GET /api/partners?account_type=company&industry_id=5

// Create new partner
POST /api/partners
{
    "account_type": "company",
    "name": "New Company Ltd",
    "email": "contact@newcompany.com",
    "tax_id": "12345678",
    "industry_id": 5,
    "tags": [1, 3, 5]
}

// Update partner
PUT /api/partners/{id}

// Get partner with relationships
GET /api/partners/{id}?include=addresses,contacts,bankAccounts,tags

// Delete partner (soft delete)
DELETE /api/partners/{id}
```

#### Contact Management
```php
// List contacts for a partner
GET /api/partners/{partnerId}/contacts

// Create contact
POST /api/partners/{partnerId}/contacts
{
    "account_type": "individual",
    "name": "Jane Smith",
    "job_title": "Purchase Manager",
    "email": "jane@company.com",
    "phone": "+1234567890"
}

// Update contact
PUT /api/partners/{partnerId}/contacts/{contactId}
```

#### Address Management
```php
// List addresses for a partner
GET /api/partners/{partnerId}/addresses

// Create address
POST /api/partners/{partnerId}/addresses
{
    "account_type": "address",
    "name": "Warehouse",
    "street1": "456 Storage Ave",
    "city": "Warehouse City",
    "country_id": 1,
    "state_id": 15
}
```

#### Bank Account Management
```php
// List bank accounts
GET /api/partners/{partnerId}/bank-accounts

// Create bank account
POST /api/partners/{partnerId}/bank-accounts
{
    "account_number": "1234567890",
    "bank_id": 1,
    "is_active": true,
    "can_send_money": false
}
```

#### Search & Filtering
```php
// Advanced search
GET /api/partners/search?q=acme&filters[account_type]=company&filters[industry_id]=5

// Geographic filtering
GET /api/partners?filters[country_id]=1&filters[state_id]=15

// Tag-based filtering
GET /api/partners?filters[tags][]=1&filters[tags][]=3
```

### Relationship Tracking APIs

#### Customer Relationship Management
```php
// Get customer statistics
GET /api/partners/{id}/customer-stats
{
    "total_orders": 45,
    "total_revenue": 125000.00,
    "last_order_date": "2025-01-15",
    "payment_terms": "Net 30",
    "credit_limit": 50000.00
}

// Customer interaction history
GET /api/partners/{id}/interactions
```

#### Vendor Relationship Management
```php
// Get vendor statistics
GET /api/partners/{id}/vendor-stats
{
    "total_purchases": 23,
    "total_amount": 89000.00,
    "last_purchase_date": "2025-01-10",
    "supplier_rank": 8,
    "payment_terms": "Net 15"
}
```

### Data Export & Integration APIs

#### Bulk Operations
```php
// Bulk export
GET /api/partners/export?format=csv&filters[account_type]=company

// Bulk import
POST /api/partners/import
Content-Type: multipart/form-data
{
    "file": [CSV file],
    "mapping": {
        "column_1": "name",
        "column_2": "email",
        "column_3": "tax_id"
    }
}

// Bulk update
PATCH /api/partners/bulk
{
    "ids": [1, 2, 3],
    "updates": {
        "industry_id": 5,
        "tags": [1, 2]
    }
}
```

## Security & Permissions

### Access Control
The partners plugin integrates with the role-based permission system:

```php
// Permission gates
'partners.view'           // View partner records
'partners.create'         // Create new partners
'partners.edit'           // Edit existing partners
'partners.delete'         // Delete partners
'partners.restore'        // Restore deleted partners
'partners.force-delete'   // Permanently delete

// Resource-specific permissions
'partners.bank-accounts.manage'  // Bank account management
'partners.contacts.manage'       // Contact management
'partners.addresses.manage'      // Address management
```

### Data Privacy
- Soft deletes preserve data integrity while hiding records
- Audit trails track all modifications
- Creator tracking for accountability
- Company-scoped access for multi-tenancy

## Plugin Configuration

### Service Provider Configuration
```php
class PartnerServiceProvider extends PackageServiceProvider
{
    public static string $name = 'partners';
    
    public function configureCustomPackage(Package $package): void
    {
        $package->name(static::$name)
            ->isCore()                    // Marks as core dependency
            ->hasTranslations()           // Enables i18n
            ->hasMigrations([...])        // Registers migrations
            ->runsMigrations();           // Auto-runs migrations
    }
}
```

### Filament Integration
```php
class PartnerPlugin implements Plugin
{
    public function register(Panel $panel): void
    {
        $panel
            ->when($panel->getId() == 'admin', function (Panel $panel) {
                $panel
                    ->discoverResources(...)    // Auto-discover resources
                    ->discoverPages(...)        // Auto-discover pages
                    ->discoverClusters(...)     // Auto-discover clusters
                    ->discoverWidgets(...);     // Auto-discover widgets
            });
    }
}
```

## Usage Examples

### Creating a Complete Customer Record
```php
// 1. Create main company
$customer = Partner::create([
    'account_type' => AccountType::COMPANY,
    'name' => 'ABC Manufacturing Ltd',
    'email' => 'info@abcmfg.com',
    'tax_id' => 'TAX123456',
    'company_registry' => 'REG789012',
    'phone' => '+1-555-0123',
    'website' => 'https://www.abcmfg.com',
    'industry_id' => 13, // Manufacturing
    'company_id' => 1,
    'creator_id' => auth()->id(),
]);

// 2. Add headquarters address
$headquarters = Partner::create([
    'account_type' => AccountType::ADDRESS,
    'parent_id' => $customer->id,
    'name' => 'Headquarters',
    'street1' => '100 Industrial Blvd',
    'street2' => 'Suite 200',
    'city' => 'Manufacturing City',
    'zip' => '12345',
    'state_id' => 15,
    'country_id' => 1,
    'creator_id' => auth()->id(),
]);

// 3. Add shipping address
$warehouse = Partner::create([
    'account_type' => AccountType::ADDRESS,
    'parent_id' => $customer->id,
    'name' => 'Shipping Warehouse',
    'street1' => '200 Logistics Way',
    'city' => 'Distribution Center',
    'zip' => '12346',
    'state_id' => 15,
    'country_id' => 1,
    'creator_id' => auth()->id(),
]);

// 4. Add primary contact
$contact1 = Partner::create([
    'account_type' => AccountType::INDIVIDUAL,
    'parent_id' => $customer->id,
    'name' => 'Sarah Johnson',
    'title_id' => 4, // Ms.
    'job_title' => 'Procurement Manager',
    'email' => 'sarah.johnson@abcmfg.com',
    'phone' => '+1-555-0124',
    'mobile' => '+1-555-0125',
    'creator_id' => auth()->id(),
]);

// 5. Add secondary contact
$contact2 = Partner::create([
    'account_type' => AccountType::INDIVIDUAL,
    'parent_id' => $customer->id,
    'name' => 'Mike Rodriguez',
    'title_id' => 4, // Mr.
    'job_title' => 'Finance Director',
    'email' => 'mike.rodriguez@abcmfg.com',
    'phone' => '+1-555-0126',
    'creator_id' => auth()->id(),
]);

// 6. Add bank account
BankAccount::create([
    'partner_id' => $customer->id,
    'bank_id' => 5,
    'account_number' => '9876543210',
    'is_active' => true,
    'can_send_money' => false,
    'creator_id' => auth()->id(),
]);

// 7. Assign tags
$customer->tags()->attach([1, 3, 7]); // VIP, Manufacturing, Large Account
```

### Querying Partner Hierarchies
```php
// Get company with all related data
$customer = Partner::with([
    'addresses',
    'contacts.title',
    'bankAccounts.bank',
    'industry',
    'tags',
    'country',
    'state'
])->find($customerId);

// Get all companies in manufacturing industry
$manufacturers = Partner::where('account_type', AccountType::COMPANY)
    ->whereHas('industry', function ($query) {
        $query->where('name', 'Manufacturing');
    })
    ->with('addresses', 'contacts')
    ->get();

// Get partners by geographic location
$partnersInState = Partner::where('account_type', '!=', AccountType::ADDRESS)
    ->where('state_id', 15)
    ->orWhereHas('addresses', function ($query) {
        $query->where('state_id', 15);
    })
    ->get();
```

## Performance Considerations

### Database Indexes
The partners table includes strategic indexes on:
- `name` - For name-based searches
- `email` - For email lookups
- `tax_id` - For tax ID searches
- `phone` and `mobile` - For phone number searches
- `company_registry` - For company registry lookups
- `reference` - For internal reference searches
- `sub_type` - For partner type filtering

### Query Optimization
- Use eager loading for related data to avoid N+1 queries
- Implement proper pagination for large datasets
- Consider database partitioning for very large partner datasets
- Use database views for complex reporting queries

### Caching Strategy
- Cache frequently accessed partner data
- Implement tag-based cache invalidation
- Use Redis for session-based partner selections
- Cache industry and title lookups

## Migration Strategy

### From Legacy CRM Systems
```php
// Example migration from legacy system
class MigrateLegacyCRMData extends Migration
{
    public function up()
    {
        // 1. Import companies
        DB::table('legacy_companies')->chunk(1000, function ($companies) {
            foreach ($companies as $company) {
                Partner::create([
                    'account_type' => AccountType::COMPANY,
                    'name' => $company->company_name,
                    'email' => $company->email,
                    'tax_id' => $company->tax_number,
                    'phone' => $company->phone,
                    'website' => $company->website,
                    // Map other fields...
                ]);
            }
        });
        
        // 2. Import contacts and link to companies
        // 3. Import addresses and link to companies
        // 4. Import bank accounts
        // 5. Create tag mappings
    }
}
```

### Version Compatibility
- Backwards compatible with existing partner references
- Graceful handling of missing relationships
- Migration scripts for schema updates
- Data validation during upgrades

## Testing Strategy

### Unit Tests
```php
class PartnerTest extends TestCase
{
    /** @test */
    public function it_can_create_a_partner_with_hierarchy()
    {
        $company = Partner::factory()->company()->create();
        $contact = Partner::factory()->individual()->create([
            'parent_id' => $company->id
        ]);
        
        $this->assertTrue($contact->parent->is($company));
        $this->assertTrue($company->contacts->contains($contact));
    }
    
    /** @test */
    public function bank_account_inherits_partner_name()
    {
        $partner = Partner::factory()->create(['name' => 'Test Company']);
        $bankAccount = BankAccount::factory()->create([
            'partner_id' => $partner->id
        ]);
        
        $this->assertEquals('Test Company', $bankAccount->account_holder_name);
    }
}
```

### Integration Tests
```php
class PartnerIntegrationTest extends TestCase
{
    /** @test */
    public function sales_order_can_reference_partner()
    {
        $customer = Partner::factory()->company()->create();
        $order = SalesOrder::factory()->create([
            'partner_id' => $customer->id
        ]);
        
        $this->assertTrue($order->partner->is($customer));
    }
}
```

## Future Enhancements

### Planned Features
1. **Advanced Communication Integration**
   - Email marketing integration
   - Social media profile linking
   - Communication preference management

2. **Enhanced Geographic Features**
   - GPS coordinates for addresses
   - Territory management
   - Distance-based partner assignment

3. **AI-Powered Features**
   - Duplicate partner detection
   - Partner scoring and ranking
   - Automated data enrichment

4. **Advanced Reporting**
   - Partner relationship analytics
   - Geographic distribution reports
   - Industry trend analysis

5. **Mobile Integration**
   - Partner mobile app
   - Field representative tools
   - Offline synchronization

## Conclusion

The Partners plugin serves as the cornerstone of the CIUIS ERP CRM system, providing a robust, flexible, and extensible foundation for managing business relationships. Its hierarchical architecture, comprehensive data model, and extensive integration capabilities make it an essential component for any ERP implementation requiring sophisticated partner management.

The plugin's design allows for seamless integration with other ERP modules while maintaining data integrity and providing the flexibility needed for complex business scenarios. Its support for multiple account types, hierarchical relationships, and extensive customization options ensures it can adapt to diverse business requirements.