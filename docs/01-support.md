# Support Plugin Documentation

## Overview

The Support plugin serves as the foundational infrastructure plugin for the CiuisERP system. It provides core entities, services, and functionality that other plugins depend on, including company management, geographic data, currency handling, plugin management, activity planning, unit of measure management, and UTM tracking capabilities.

## Database Schema

### Core Entity Tables

#### 1. Companies (`companies`)
Central entity representing business organizations within the system.

```sql
CREATE TABLE companies (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    parent_id BIGINT NULL,                    -- Self-referencing for company hierarchies
    currency_id BIGINT NULL,                  -- Default currency for company
    creator_id BIGINT NULL,                   -- User who created the company
    partner_id BIGINT NOT NULL,               -- Links to partners_partners table
    sort INT NULL,                           -- Sortable order
    name VARCHAR(255) NOT NULL,              -- Company name
    company_id VARCHAR(255) NULL UNIQUE,     -- External company identifier
    tax_id VARCHAR(255) NULL UNIQUE,         -- Tax identification number
    registration_number VARCHAR(255) NULL,   -- Business registration number
    email VARCHAR(255) NULL,                 -- Primary contact email
    phone VARCHAR(255) NULL,                 -- Primary phone
    mobile VARCHAR(255) NULL,                -- Mobile phone
    website VARCHAR(255) NULL,               -- Company website
    color VARCHAR(255) NULL,                 -- UI color theme
    is_active BOOLEAN DEFAULT true,          -- Active status
    founded_date DATE NULL,                  -- Date company was founded
    -- Address fields
    street1 VARCHAR(255) NULL,               -- Primary street address
    street2 VARCHAR(255) NULL,               -- Secondary street address
    city VARCHAR(255) NULL,                  -- City
    zip VARCHAR(255) NULL,                   -- Postal code
    state_id BIGINT NULL,                    -- State/province reference
    country_id BIGINT NULL,                  -- Country reference
    deleted_at TIMESTAMP NULL,               -- Soft delete
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Business Rules:**
- Companies can have parent-child relationships (branches)
- Each company automatically creates a corresponding Partner record
- Supports multi-company operations with user access control
- Integrates with Chatter for communication tracking
- Supports custom fields through HasCustomFields trait

#### 2. Currencies (`currencies`)
Currency definitions for financial operations.

```sql
CREATE TABLE currencies (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,              -- Currency code (USD, EUR, etc.)
    symbol VARCHAR(255) NULL,                -- Currency symbol ($, €, etc.)
    iso_numeric INT NULL,                    -- ISO 4217 numeric code
    decimal_places TINYINT NULL,             -- Number of decimal places
    full_name VARCHAR(255) NULL,             -- Full currency name
    rounding DECIMAL(8,2) DEFAULT 0.00,      -- Rounding precision
    active BOOLEAN DEFAULT true,             -- Active status
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 3. Countries (`countries`)
Geographic country data with currency associations.

```sql
CREATE TABLE countries (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    currency_id BIGINT NULL,                 -- Default currency for country
    phone_code VARCHAR(255) NULL,            -- International dialing code
    code VARCHAR(2) NULL,                    -- ISO country code
    name VARCHAR(255) NULL,                  -- Country name
    state_required BOOLEAN DEFAULT false,    -- Whether state is required for addresses
    zip_required BOOLEAN DEFAULT false,      -- Whether zip code is required
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 4. States (`states`)
State/province data for countries.

```sql
CREATE TABLE states (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    country_id BIGINT NOT NULL,              -- Parent country
    name VARCHAR(255) NOT NULL,              -- State/province name
    code VARCHAR(255) NOT NULL,              -- State/province code
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 5. Banks (`banks`)
Financial institution data.

```sql
CREATE TABLE banks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NULL,                  -- Bank name
    code VARCHAR(255) NULL,                  -- Bank code/identifier
    email VARCHAR(255) NULL,                 -- Bank contact email
    phone VARCHAR(255) NULL,                 -- Bank phone
    street1 VARCHAR(255) NULL,               -- Bank address
    street2 VARCHAR(255) NULL,
    city VARCHAR(255) NULL,
    zip VARCHAR(255) NULL,
    state_id BIGINT NULL,                    -- Bank state
    country_id BIGINT NULL,                  -- Bank country
    creator_id BIGINT NOT NULL,              -- User who created bank record
    deleted_at TIMESTAMP NULL,               -- Soft delete
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### Plugin Management Tables

#### 6. Plugins (`plugins`)
Plugin registry and management.

```sql
CREATE TABLE plugins (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) UNIQUE NOT NULL,       -- Plugin identifier
    author VARCHAR(255) NULL,                -- Plugin author
    summary TEXT NULL,                       -- Short description
    description TEXT NULL,                   -- Detailed description
    latest_version VARCHAR(255) NULL,        -- Current version
    license VARCHAR(255) NULL,               -- License type
    is_active BOOLEAN DEFAULT false,         -- Active status
    is_installed BOOLEAN DEFAULT false,      -- Installation status
    sort INT NULL,                          -- Display order
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 7. Plugin Dependencies (`plugin_dependencies`)
Plugin dependency management.

```sql
CREATE TABLE plugin_dependencies (
    plugin_id BIGINT NOT NULL,               -- Plugin requiring dependency
    dependency_id BIGINT NOT NULL,           -- Required plugin
    UNIQUE(plugin_id, dependency_id)
);
```

### Activity Management Tables

#### 8. Activity Plans (`activity_plans`)
Template-based activity planning system.

```sql
CREATE TABLE activity_plans (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    plugin VARCHAR(255) NULL,                -- Originating plugin
    name VARCHAR(255) NOT NULL,              -- Plan name
    is_active BOOLEAN DEFAULT false,         -- Active status
    creator_id BIGINT NULL,                  -- Plan creator
    company_id BIGINT NULL,                  -- Associated company
    deleted_at TIMESTAMP NULL,               -- Soft delete
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 9. Activity Types (`activity_types`)
Configurable activity type definitions.

```sql
CREATE TABLE activity_types (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sort INT NULL,                          -- Display order
    delay_count INT NULL,                   -- Delay before execution
    delay_unit VARCHAR(255) NOT NULL,       -- Delay unit (minutes, hours, days, weeks)
    delay_from VARCHAR(255) NOT NULL,       -- Delay reference point
    icon VARCHAR(255) NULL,                 -- Display icon
    decoration_type VARCHAR(255) NULL,      -- Visual decoration
    chaining_type VARCHAR(255) DEFAULT 'suggest', -- How activities chain (suggest/trigger)
    plugin VARCHAR(255) NULL,               -- Originating plugin
    category VARCHAR(255) NULL,             -- Activity category
    name VARCHAR(255) NOT NULL,             -- Activity type name
    summary TEXT NULL,                      -- Description
    default_note TEXT NULL,                 -- Default note template
    is_active BOOLEAN DEFAULT true,         -- Active status
    keep_done BOOLEAN DEFAULT false,        -- Preserve completed activities
    creator_id BIGINT NULL,                 -- Creator user
    default_user_id BIGINT NULL,            -- Default assignee
    activity_plan_id BIGINT NULL,           -- Parent plan
    triggered_next_type_id BIGINT NULL,     -- Auto-triggered next activity
    deleted_at TIMESTAMP NULL,              -- Soft delete
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 10. Activity Plan Templates (`activity_plan_templates`)
Template configurations for activity plans.

```sql
CREATE TABLE activity_plan_templates (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sort INT NULL,                          -- Template order
    plan_id BIGINT NOT NULL,                -- Parent plan
    activity_type_id BIGINT NOT NULL,       -- Activity type
    responsible_id BIGINT NULL,             -- Responsible user
    creator_id BIGINT NULL,                 -- Template creator
    delay_count INT NULL,                   -- Template delay
    delay_unit VARCHAR(255) NOT NULL,       -- Delay unit
    delay_from VARCHAR(255) NOT NULL,       -- Delay reference
    summary TEXT NULL,                      -- Template summary
    responsible_type VARCHAR(255) NOT NULL, -- Responsibility assignment type
    note TEXT NULL,                         -- Template note
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 11. Activity Type Suggestions (`activity_type_suggestions`)
Activity type recommendation system.

```sql
CREATE TABLE activity_type_suggestions (
    activity_type_id BIGINT NOT NULL,       -- Source activity type
    suggested_activity_type_id BIGINT NOT NULL -- Suggested next activity type
);
```

### Unit of Measure Tables

#### 12. Unit of Measure Categories (`unit_of_measure_categories`)
Grouping for related units of measure.

```sql
CREATE TABLE unit_of_measure_categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,              -- Category name (Weight, Volume, Length, etc.)
    creator_id BIGINT NULL,                  -- Creator user
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 13. Unit of Measures (`unit_of_measures`)
Conversion-capable measurement units.

```sql
CREATE TABLE unit_of_measures (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    type VARCHAR(255) NOT NULL,              -- UOM type (reference, bigger, smaller)
    name VARCHAR(255) NOT NULL,              -- Unit name (kg, gram, meter, etc.)
    factor DECIMAL(15,4) DEFAULT 0,          -- Conversion factor
    rounding DECIMAL(15,4) DEFAULT 0,        -- Rounding precision
    category_id BIGINT NOT NULL,             -- Parent category
    creator_id BIGINT NULL,                  -- Creator user
    deleted_at TIMESTAMP NULL,               -- Soft delete
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### UTM Tracking Tables

#### 14. UTM Sources (`utm_sources`)
Traffic source tracking.

```sql
CREATE TABLE utm_sources (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,              -- Source name (google, facebook, email, etc.)
    creator_id BIGINT NULL,                  -- Creator user
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 15. UTM Mediums (`utm_mediums`)
Marketing medium tracking.

```sql
CREATE TABLE utm_mediums (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,              -- Medium name (cpc, email, social, etc.)
    creator_id BIGINT NULL,                  -- Creator user
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 16. UTM Stages (`utm_stages`)
Campaign stage management.

```sql
CREATE TABLE utm_stages (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sort INT NULL,                          -- Stage order
    name VARCHAR(255) NOT NULL,              -- Stage name
    created_by BIGINT NULL,                  -- Creator user
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### 17. UTM Campaigns (`utm_campaigns`)
Marketing campaign tracking.

```sql
CREATE TABLE utm_campaigns (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NULL,                     -- Responsible user
    stage_id BIGINT NOT NULL,                -- Campaign stage
    color VARCHAR(255) NULL,                 -- UI color
    created_by BIGINT NULL,                  -- Creator user
    name VARCHAR(255) NOT NULL,              -- Campaign identifier
    title VARCHAR(255) NOT NULL,             -- Campaign display name
    is_active BOOLEAN DEFAULT false,         -- Active status
    is_auto_campaign BOOLEAN DEFAULT false, -- Automated campaign flag
    company_id BIGINT NULL,                  -- Associated company
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### Utility Tables

#### 18. User Allowed Companies (`user_allowed_companies`)
Multi-company access control.

```sql
CREATE TABLE user_allowed_companies (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,                 -- User reference
    company_id BIGINT NOT NULL               -- Allowed company
);
```

#### 19. Email Logs (`email_logs`)
Email communication tracking.

```sql
CREATE TABLE email_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    recipient_email VARCHAR(255) NOT NULL,   -- Recipient email
    recipient_name VARCHAR(255) NOT NULL,    -- Recipient name
    subject VARCHAR(255) NOT NULL,           -- Email subject
    status VARCHAR(255) NOT NULL,            -- Send status
    error_message TEXT NULL,                 -- Error details if failed
    sent_at TIMESTAMP NOT NULL,              -- Send timestamp
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

## Business Logic & Models

### Core Models

#### Company Model
**File:** `/plugins/webkul/support/src/Models/Company.php`

**Key Features:**
- Hierarchical company structure (parent/child relationships)
- Automatic Partner record creation and synchronization
- Geographic location support with country/state relationships
- Currency association for financial operations
- Sortable with custom ordering
- Soft deletion support
- Chatter integration for communication tracking
- Custom fields support
- Multi-company user access control

**Key Methods:**
- `isBranch()`: Check if company is a subsidiary
- `isParent()`: Check if company is a parent organization
- `scopeParents()`: Query scope for parent companies only
- `scopeActive()`: Query scope for active companies

**Relationships:**
- `parent()`: Parent company (self-referencing)
- `branches()`: Child companies
- `currency()`: Default currency
- `country()`: Geographic country
- `state()`: Geographic state
- `createdBy()`: User who created the company
- `partner()`: Associated partner record

#### Currency Model
**File:** `/plugins/webkul/support/src/Models/Currency.php`

**Purpose:** Manages currency definitions with ISO standards support and rounding precision for financial calculations.

#### Country/State Models
**Files:** `/plugins/webkul/support/src/Models/Country.php`, `/plugins/webkul/support/src/Models/State.php`

**Purpose:** Hierarchical geographic data with address validation rules and currency associations.

#### Plugin Model
**File:** `/plugins/webkul/support/src/Models/Plugin.php`

**Key Features:**
- Plugin dependency management with many-to-many relationships
- Installation and activation status tracking
- Version management
- Sortable plugin ordering

**Relationships:**
- `dependencies()`: Required plugins
- `dependents()`: Plugins that depend on this one

#### Activity Management Models

##### ActivityPlan Model
**File:** `/plugins/webkul/support/src/Models/ActivityPlan.php`

**Purpose:** Template-based activity planning system that allows plugins to define automated activity sequences.

**Features:**
- Plugin-specific activity plans
- Company-specific customization
- Custom fields support

##### ActivityType Model
**File:** `/plugins/webkul/support/src/Models/ActivityType.php`

**Purpose:** Configurable activity definitions with timing, chaining, and automation capabilities.

**Key Features:**
- Delay configuration (count, unit, reference point)
- Activity chaining (suggest vs trigger)
- Visual customization (icons, decorations)
- Plugin categorization
- Self-referencing trigger relationships
- Activity suggestions system

#### UOM (Unit of Measure) Models

##### UOM Model
**File:** `/plugins/webkul/support/src/Models/UOM.php`

**Key Features:**
- Category-based unit organization
- Conversion factor calculations
- Precision rounding support
- Cross-category conversion validation

**Key Methods:**
- `computeQuantity()`: Convert quantities between units with validation and rounding
- `floatRound()`: Custom rounding implementation with multiple strategies

### Enums

#### ActivityChainingType
**Values:** `suggest`, `trigger`
**Purpose:** Defines how activities link together in workflows

#### ActivityDelayUnit
**Values:** `minutes`, `hours`, `days`, `weeks`
**Purpose:** Time unit specifications for activity scheduling

#### UOMType
**Values:** `reference`, `bigger`, `smaller`
**Purpose:** Unit of measure classification for conversion calculations

## Plugin Architecture

### Plugin Registration System

The Support plugin implements a sophisticated plugin management system:

#### PluginManager
**File:** `/plugins/webkul/support/src/PluginManager.php`

**Purpose:** Dynamically discovers and registers Filament plugins from the bootstrap configuration.

**Process:**
1. Reads plugin list from `/bootstrap/plugins.php`
2. Instantiates each plugin class
3. Registers plugins with the Filament panel

#### SupportPlugin
**File:** `/plugins/webkul/support/src/SupportPlugin.php`

**Features:**
- Auto-discovery of Filament resources, pages, clusters, and widgets
- Panel-specific configuration (admin vs customer panels)
- JavaScript injection for UI enhancements
- Sidebar navigation improvements

### Service Provider Architecture

#### SupportServiceProvider
**File:** `/plugins/webkul/support/src/SupportServiceProvider.php`

**Extends:** `PackageServiceProvider` (custom base class)

**Key Features:**
- Core plugin designation (`isCore()`)
- Comprehensive migration management
- Translation support
- Command registration
- Event listener setup
- Route registration for image caching
- Policy registration
- Livewire component registration

## Entity Relationships

### Primary Relationships

```
Companies (1) ←→ (n) Companies [parent/child hierarchy]
Companies (n) ←→ (1) Currencies [default currency]
Companies (n) ←→ (1) Countries [location]
Companies (n) ←→ (1) States [location]
Companies (1) ←→ (1) Partners [business entity sync]

Countries (1) ←→ (n) States [geographic hierarchy]
Countries (n) ←→ (1) Currencies [default currency]

Plugins (n) ←→ (n) Plugins [dependencies]

ActivityPlans (1) ←→ (n) ActivityTypes [plan templates]
ActivityPlans (n) ←→ (1) Companies [company-specific plans]
ActivityTypes (n) ←→ (n) ActivityTypes [suggestions]
ActivityTypes (1) ←→ (n) ActivityTypes [trigger chains]

UOMCategories (1) ←→ (n) UOMs [measurement grouping]

UTMCampaigns (n) ←→ (1) UTMStages [campaign lifecycle]
UTMCampaigns (n) ←→ (1) Companies [campaign ownership]

Users (n) ←→ (n) Companies [access control via user_allowed_companies]
```

### Cross-Plugin Dependencies

The Support plugin provides foundation services that other plugins depend on:

1. **Company Management**: All business operations require company context
2. **Geographic Data**: Address handling across all modules
3. **Currency Handling**: Financial calculations and display
4. **Activity Planning**: Workflow automation for business processes
5. **Unit of Measure**: Product management and inventory operations
6. **UTM Tracking**: Marketing and analytics across modules

## API Requirements

### Core Entity Endpoints

#### Company Management
```http
GET    /api/companies                 # List companies with filtering
POST   /api/companies                 # Create new company
GET    /api/companies/{id}            # Get company details
PUT    /api/companies/{id}            # Update company
DELETE /api/companies/{id}            # Soft delete company
GET    /api/companies/{id}/branches   # Get company branches
POST   /api/companies/{id}/branches   # Create branch company
```

#### Geographic Data
```http
GET    /api/countries                 # List countries with currency data
GET    /api/countries/{id}/states     # Get states for country
GET    /api/states/{id}               # Get state details
```

#### Currency Operations
```http
GET    /api/currencies                # List active currencies
GET    /api/currencies/{id}/rate      # Get current exchange rate
POST   /api/currencies/convert        # Convert between currencies
```

#### Plugin Management
```http
GET    /api/plugins                   # List installed plugins
POST   /api/plugins/{id}/activate     # Activate plugin
POST   /api/plugins/{id}/deactivate   # Deactivate plugin
GET    /api/plugins/{id}/dependencies # Get plugin dependencies
```

#### Activity Planning
```http
GET    /api/activity-plans            # List activity plans
POST   /api/activity-plans            # Create activity plan
GET    /api/activity-plans/{id}/types # Get activity types for plan
POST   /api/activity-types            # Create activity type
PUT    /api/activity-types/{id}       # Update activity type
GET    /api/activity-types/{id}/suggestions # Get suggested next activities
```

#### Unit of Measure
```http
GET    /api/uom-categories            # List UOM categories
GET    /api/uoms                      # List units of measure
POST   /api/uoms/convert              # Convert quantities between units
GET    /api/uoms/{categoryId}         # Get UOMs in category
```

#### UTM Tracking
```http
GET    /api/utm/sources               # List UTM sources
GET    /api/utm/mediums               # List UTM mediums
GET    /api/utm/campaigns             # List campaigns with filtering
POST   /api/utm/campaigns             # Create campaign
PUT    /api/utm/campaigns/{id}        # Update campaign
GET    /api/utm/campaigns/{id}/stats  # Get campaign statistics
```

### Utility Endpoints

#### Email Management
```http
GET    /api/email-logs                # List email logs with filtering
GET    /api/email-logs/{id}           # Get email log details
POST   /api/emails/send               # Send email with logging
```

#### Bank Information
```http
GET    /api/banks                     # List banks with filtering
POST   /api/banks                     # Create bank record
PUT    /api/banks/{id}                # Update bank
DELETE /api/banks/{id}                # Soft delete bank
```

#### Multi-Company Access
```http
GET    /api/users/{id}/companies      # Get user's allowed companies
POST   /api/users/{id}/companies      # Grant company access
DELETE /api/users/{id}/companies/{companyId} # Revoke company access
```

### Advanced API Features

#### Filtering and Search
All list endpoints should support:
- `?search=term` - Full-text search
- `?filter[field]=value` - Field-specific filtering
- `?sort=field` - Sorting by field
- `?include=relation1,relation2` - Eager loading
- `?page=1&per_page=25` - Pagination

#### Bulk Operations
```http
POST   /api/companies/bulk            # Bulk create/update companies
DELETE /api/companies/bulk            # Bulk delete companies
POST   /api/uoms/bulk-convert         # Bulk quantity conversions
```

#### Export/Import
```http
GET    /api/companies/export          # Export companies (CSV/Excel)
POST   /api/companies/import          # Import companies
GET    /api/currencies/export         # Export currency data
```

## Helper Functions

### Money Formatting
**File:** `/plugins/webkul/support/src/helpers.php`

```php
money(float $amount, string $currency = null, int $divideBy = 0, string $locale = null): string
```

**Purpose:** Standardized currency formatting across the application with locale support.

## Security Considerations

1. **Multi-Company Data Isolation**: Ensure users can only access data from their allowed companies
2. **Plugin Dependency Validation**: Prevent installation of plugins with missing dependencies
3. **Soft Deletion**: Preserve data integrity with soft deletes for audit trails
4. **Role-Based Access**: Integrate with the Security plugin for proper authorization
5. **Input Validation**: Validate all geographic data, currency codes, and UOM conversions
6. **Rate Limiting**: Implement rate limiting on conversion and bulk operation endpoints

## Integration Points

### Partner Plugin Integration
- Automatic Partner record creation for each Company
- Bidirectional synchronization of company and partner data
- Shared address and contact information

### Security Plugin Integration
- User authentication and authorization
- Role-based access control for company data
- Permission-based plugin management

### Field Plugin Integration
- Custom fields support for Companies and Activity Plans
- Dynamic field definitions and validation

### Chatter Plugin Integration
- Communication tracking for Company records
- Activity logging and discussion threads

This Support plugin forms the critical foundation that enables the modular, multi-company ERP system architecture while providing essential services for geographic data management, currency handling, workflow automation, and marketing analytics.