# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Installation & Setup
```bash
# Complete ERP installation (migrations, seeders, admin setup)
php artisan erp:install

# Install specific plugins
php artisan {plugin-name}:install
# Examples: php artisan inventories:install, php artisan accounts:install

# Uninstall plugins
php artisan {plugin-name}:uninstall
```

### Development Workflow
```bash
# Start complete development environment (server, queue, logs, vite concurrently)
composer dev

# Individual development processes
php artisan serve              # Laravel development server
php artisan queue:listen       # Queue worker
php artisan pail              # Real-time log monitoring
npm run dev                   # Vite development server

# Frontend build
npm run build                 # Production build
```

### Code Quality & Testing
```bash
# Code formatting (Laravel Pint with custom binary operator alignment)
./vendor/bin/pint             # Format all files following Laravel preset
./vendor/bin/pint --test      # Check formatting without making changes

# Testing
./vendor/bin/phpunit          # Run all tests
./vendor/bin/phpunit --group feature
./vendor/bin/phpunit --group unit
```

### Database Operations
```bash
# Standard Laravel database commands work across all plugins
php artisan migrate
php artisan db:seed
php artisan make:factory ModelFactory
```

## Architecture Overview

### Plugin-Based Modular System
AureusERP uses a sophisticated plugin architecture where functionality is organized into self-contained modules under `/plugins/webkul/`. The system automatically discovers and loads plugins through:

1. **Plugin Registry**: `/bootstrap/plugins.php` - Active plugin list
2. **Plugin Manager**: `/plugins/webkul/support/src/PluginManager.php` - Dynamic plugin loader
3. **Composer Merge**: Uses `wikimedia/composer-merge-plugin` to merge all plugin composer.json files
4. **Service Providers**: Each plugin extends `PackageServiceProvider` for consistent initialization
5. **Filament Integration**: Plugins register admin panel resources through Filament plugin system

### Plugin Categories

**Core Plugins** (Essential system components, installed by default):
- **Analytics**: Business intelligence and reporting
- **Chatter**: Internal communication platform
- **Fields**: Customizable data structure management
- **Security**: Role-based access control
- **Support**: Help desk and documentation system
- **Table Views**: Customizable data presentation

**Installable Plugins** (Install as needed):
- **Accounts**: Financial accounting with complex tax/fiscal management
- **Blogs**: Content management system
- **Contacts/Partners**: Customer/vendor relationship management
- **Employees**: HR management with departments, skills, calendars
- **Inventories**: Warehouse management with locations, operations, routes
- **Products**: Product catalog management
- **Projects**: Project planning and management
- **Purchases/Sales**: Procurement and sales pipeline management
- **Invoices/Payments**: Financial transaction processing
- **Timesheets/Time-off**: Work hour tracking and leave management
- **Website**: Customer-facing website functionality

### Plugin Structure Pattern
Each plugin follows consistent structure:
```
plugins/webkul/{plugin-name}/
├── composer.json                    # Plugin dependencies
├── database/
│   ├── factories/                  # Model factories
│   ├── migrations/                 # Database migrations
│   └── seeders/                    # Database seeders
├── resources/lang/                 # Translations
├── src/
│   ├── {Name}Plugin.php           # Filament plugin registration
│   ├── {Name}ServiceProvider.php  # Laravel service provider
│   ├── Models/                     # Eloquent models
│   ├── Filament/                   # Admin panel resources
│   ├── Policies/                   # Authorization policies
│   ├── Enums/                      # PHP enums
│   └── Traits/                     # Reusable traits
```

### Technology Stack
- **Backend**: Laravel 11.x (PHP 8.2+)
- **Admin Panel**: FilamentPHP 3.x with Filament Shield for permissions
- **Frontend**: Vite + TailwindCSS
- **Database**: MySQL 8.0+ or SQLite
- **Queue**: Laravel queue system with concurrent worker
- **Real-time Logging**: Laravel Pail for development
- **Code Style**: Laravel Pint with custom binary operator alignment

### Key Integration Points

**Plugin Dependencies**: Plugins can declare dependencies on other plugins. Installation commands automatically handle dependency resolution and prompt for conflicts.

**Database Management**: Each plugin manages its own migrations, seeders, and settings with proper namespacing. All standard Laravel database commands work across plugins.

**Filament Admin Integration**: Each plugin registers its admin resources through the Filament plugin system. The PluginManager automatically discovers and loads all active plugins.

**Permission System**: Uses Filament Shield for automatic role/permission generation across all plugin resources.

### Development Notes

- Plugin autoloading is handled through composer.json merging
- The `composer dev` script runs all development services concurrently
- Code style is enforced with Laravel Pint using Laravel preset + custom binary operator alignment
- Each plugin is completely self-contained but integrates through the core Laravel/Filament framework
- Plugin installation/uninstallation includes dependency checking and data seeding options