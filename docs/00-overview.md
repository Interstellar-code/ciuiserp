# AureusERP System Overview

This document provides a comprehensive overview of the AureusERP system architecture, plugin structure, and system-wide relationships.

## 📋 Table of Contents

1. [System Architecture](#system-architecture)
2. [Plugin Catalog](#plugin-catalog)
3. [Technology Stack](#technology-stack)
4. [Plugin Dependencies](#plugin-dependencies)
5. [Cross-Plugin Relationships](#cross-plugin-relationships)
6. [Development Workflow](#development-workflow)
7. [Migration to ShadCN/React](#migration-to-shadcnreact)

## System Architecture

AureusERP is built as a **modular, plugin-based ERP system** using Laravel 11.x and FilamentPHP 3.x with a sophisticated auto-discovery mechanism for plugins.

### Core Architecture Components

```
├── Core Laravel App
│   ├── Authentication & Authorization
│   ├── Multi-tenant Company System
│   └── Base Service Providers
├── Plugin Manager (Webkul\Support\PluginManager)
│   ├── Auto-discovery via bootstrap/plugins.php
│   ├── Composer merge plugin integration
│   └── Filament resource registration
└── Individual Plugins (22 total)
    ├── Core Plugins (essential)
    └── Business Plugins (installable)
```

### Multi-Tenant Design

```sql
-- Core multi-tenancy pattern
companies (id, name, settings, ...)
├── All business entities belong to a company
├── user_allowed_companies (user-company access)
└── Company-based data isolation
```

## Plugin Catalog

### 🔧 Core Plugins (Essential System Components)

| Plugin | Purpose | Status | Tables |
|--------|---------|--------|---------|
| **support** | Core infrastructure (companies, currencies, countries, etc.) | ✅ Active | 15+ tables |
| **security** | Role-based access control & authentication | ✅ Active | Extends Laravel permissions |
| **fields** | Custom field system for all entities | ✅ Active | 1 table + polymorphic |
| **chatter** | Internal communication & collaboration | ✅ Active | 3 tables |
| **analytics** | Business intelligence & reporting | ✅ Active | 1 table |
| **table-views** | Customizable data presentation | ✅ Active | Configuration-based |

### 💼 Business Plugins (Installable Modules)

#### Financial Management
| Plugin | Purpose | Status | Tables | Dependencies |
|--------|---------|--------|---------|-------------|
| **accounts** | Financial accounting, tax management, payments | ✅ Active | 40+ tables | support, partners, products |
| **invoices** | Invoice generation & management | ✅ Active | TBD | accounts, partners |
| **payments** | Payment processing & tracking | ✅ Active | TBD | accounts |

#### Customer & Partner Management  
| Plugin | Purpose | Status | Tables | Dependencies |
|--------|---------|--------|---------|-------------|
| **partners** | CRM - customers, vendors, contacts | ✅ Active | 10+ tables | support |
| **contacts** | Extended contact management | ✅ Active | TBD | partners |

#### Human Resources
| Plugin | Purpose | Status | Tables | Dependencies |
|--------|---------|--------|---------|-------------|
| **employees** | HR management, departments, skills | ✅ Active | 15+ tables | support, partners |
| **recruitments** | Applicant tracking & hiring | ✅ Active | TBD | employees |
| **time-off** | Leave management & tracking | ✅ Active | TBD | employees |
| **timesheets** | Work hour tracking | ✅ Active | TBD | employees |

#### Product & Inventory Management
| Plugin | Purpose | Status | Tables | Dependencies |
|--------|---------|--------|---------|-------------|
| **products** | Product catalog & management | ✅ Active | 20+ tables | support |
| **inventories** | Warehouse & stock management | ✅ Active | 25+ tables | support, products |

#### Sales & Procurement
| Plugin | Purpose | Status | Tables | Dependencies |
|--------|---------|--------|---------|-------------|
| **sales** | Sales pipeline & opportunity management | ✅ Active | TBD | partners, products |
| **purchases** | Procurement & purchase orders | ✅ Active | TBD | partners, products |

#### Project & Content Management
| Plugin | Purpose | Status | Tables | Dependencies |
|--------|---------|--------|---------|-------------|
| **projects** | Project planning & management | ✅ Active | TBD | partners, employees |
| **blogs** | Content management system | ✅ Active | 4 tables | support |
| **website** | Customer-facing website | ✅ Active | TBD | various |

## Technology Stack

### Backend (Current)
- **Laravel 11.x** - PHP framework
- **FilamentPHP 3.x** - Admin panel framework
- **Spatie Permissions** - Role-based access control
- **Laravel Pail** - Real-time logging
- **Composer Merge Plugin** - Plugin auto-discovery

### Frontend (Target - ShadCN/React)
- **React 18+** - Frontend framework  
- **ShadCN/UI** - Component library
- **TailwindCSS** - Styling
- **Vite** - Build tool
- **Laravel Sanctum** - API authentication

### Database
- **MySQL 8.0+** or **SQLite** - Primary database
- **Redis** - Cache & sessions (optional)

## Plugin Dependencies

### Dependency Tree
```
Level 0 (Foundation):
├── support (companies, currencies, countries, etc.)
└── security (roles, permissions)

Level 1 (Core Business):  
├── partners (requires: support)
├── products (requires: support)
└── employees (requires: support, partners)

Level 2 (Business Logic):
├── accounts (requires: support, partners, products)
├── inventories (requires: support, products)
└── sales/purchases (requires: partners, products)

Level 3 (Advanced Features):
├── projects (requires: partners, employees)
├── timesheets (requires: employees)
└── website (requires: various)
```

### Critical Integration Points

#### Partner-Centric Integration
- **Employee → Partner**: Every employee auto-creates a partner record
- **Accounts → Partner**: Customer/vendor accounting relationships
- **Sales/Purchases → Partner**: Commercial relationships

#### Product-Centric Integration  
- **Products → Accounts**: Tax configuration and cost accounting
- **Products → Inventories**: Stock tracking and movements
- **Products → Sales/Purchases**: Commercial transactions

#### Financial Integration
- **Accounts → All Modules**: Financial impact tracking
- **Payments → Invoices**: Payment allocation
- **Inventories → Accounts**: Stock valuation (future)

## Cross-Plugin Relationships

### Database-Level Relationships

```sql
-- Core multi-tenant relationship
support_companies
├── partners_partners.company_id
├── employees_employees.company_id  
├── products_products.company_id
└── [all business entities].company_id

-- Partner integration pattern
partners_partners
├── employees_employees.partner_id (auto-created)
├── accounts_account_moves.partner_id
└── sales/purchase transactions.partner_id

-- Product integration pattern  
products_products
├── accounts_product_taxes (tax configuration)
├── inventories_product_quantities (stock levels)
└── sales/purchase lines.product_id
```

### Business Process Integration

#### Employee Creation Workflow
1. Employee record created → Auto-creates Partner record
2. Employee gets company access → Partner inherits permissions
3. Employee bank details → Stored in Partner model

#### Sales/Purchase Workflow  
1. Partner selection (customer/vendor)
2. Product selection with tax rules
3. Accounting entries generation
4. Inventory impact (if applicable)
5. Payment processing integration

#### Inventory Operations
1. Product-based stock tracking
2. Location-based operations
3. Partner-based transfers
4. Future: Accounting integration for valuation

## Development Workflow

### Current Commands
```bash
# System installation
php artisan erp:install

# Plugin management  
php artisan {plugin}:install
php artisan {plugin}:uninstall

# Development environment
composer dev  # Concurrent server, queue, logs, vite

# Code quality
./vendor/bin/pint        # Laravel Pint formatting
./vendor/bin/phpunit     # Testing
```

### Plugin Structure
```
plugins/webkul/{plugin}/
├── composer.json                    # Dependencies
├── database/
│   ├── migrations/                  # Database schema
│   ├── seeders/                     # Sample data
│   └── factories/                   # Test data
├── src/
│   ├── {Plugin}Plugin.php           # Filament registration
│   ├── {Plugin}ServiceProvider.php  # Laravel integration
│   ├── Models/                      # Business entities
│   ├── Enums/                       # Business constants
│   ├── Filament/                    # Admin UI (to be replaced)
│   ├── Policies/                    # Authorization
│   └── Traits/                      # Reusable logic
└── resources/
    ├── lang/                        # Translations
    └── views/                       # Templates
```

## Migration to ShadCN/React

### Migration Strategy

#### Phase 1: API Foundation
1. Create Laravel API controllers for each plugin
2. Implement JWT authentication (Laravel Sanctum)
3. Preserve business logic in service classes
4. Maintain database schema and relationships

#### Phase 2: Frontend Replacement
1. Build ShadCN/React components per plugin
2. Implement plugin-by-plugin migration
3. Maintain feature parity with Filament UI
4. Add modern UX improvements

#### Phase 3: System Integration
1. Real-time features (WebSockets)
2. Advanced search and filtering
3. Mobile responsiveness
4. Performance optimization

### Plugin Migration Priority

**High Priority (Core Business)**:
1. **support** - Multi-tenant foundation
2. **security** - Authentication & authorization  
3. **partners** - CRM foundation
4. **employees** - HR basics
5. **accounts** - Financial management

**Medium Priority (Business Operations)**:
6. **products** - Product catalog
7. **inventories** - Stock management
8. **sales/purchases** - Commercial operations
9. **projects** - Project management

**Low Priority (Extended Features)**:
10. **blogs/website** - Content management
11. **chatter** - Internal communication
12. **analytics** - Business intelligence
13. **timesheets/time-off** - Advanced HR

### Architecture Benefits

#### Current Filament Limitations
- Monolithic admin interface
- Limited customization for complex workflows
- Performance constraints with large datasets
- Mobile experience limitations

#### ShadCN/React Benefits
- **Modern UX**: Responsive, mobile-first design
- **Performance**: Client-side rendering, optimized bundles
- **Customization**: Complete control over UI/UX
- **Scalability**: Better handling of complex business processes
- **Developer Experience**: Modern React ecosystem

### Key Considerations

#### Data Integrity
- Preserve all existing business logic
- Maintain database relationships
- Keep validation rules and constraints
- Ensure audit trails and permissions

#### Business Continuity  
- Plugin-by-plugin migration approach
- Maintain backward compatibility during transition
- Preserve all existing features
- Training and documentation updates

#### Performance Requirements
- Real-time updates for inventory and financial data
- Efficient handling of large datasets (products, transactions)
- Optimized search and filtering
- Mobile performance optimization

This overview provides the foundation for understanding the complete AureusERP system before diving into individual plugin documentation.