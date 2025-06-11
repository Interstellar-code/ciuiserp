# Analytics Plugin Documentation

## Overview

The Analytics plugin provides business intelligence and reporting capabilities for the CIUIS ERP system, enabling data-driven decision making through configurable analytics records and dashboard widgets.

## Database Schema

### Analytics Records Table

```sql
CREATE TABLE analytic_records (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    company_id BIGINT UNSIGNED NOT NULL,
    creator_id BIGINT UNSIGNED NOT NULL,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT 1,
    sort INTEGER NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    
    FOREIGN KEY (company_id) REFERENCES support_companies(id),
    FOREIGN KEY (creator_id) REFERENCES users(id),
    UNIQUE KEY unique_company_code (company_id, code),
    INDEX idx_active (is_active),
    INDEX idx_sort (sort)
);
```

## Business Logic

### Record Model

The `Record` model provides:
- **Multi-company support**: Analytics records are company-specific
- **Sortable interface**: Records can be ordered and prioritized
- **Active status**: Enable/disable analytics records
- **Unique codes**: Each record has a unique code within company scope

### Key Features

1. **Analytic Dimensions**: Create custom analytic dimensions for reporting
2. **Cost Center Tracking**: Track costs and revenues by department or project
3. **Performance Metrics**: Define KPIs and performance indicators
4. **Dashboard Integration**: Provide data for dashboard widgets
5. **Multi-company Analytics**: Separate analytics per company

## Entity Relationships

### Integration Points

- **All Business Modules**: Analytics records can be referenced from any module
- **Accounts Plugin**: Financial analytics and cost center tracking
- **Employees Plugin**: HR analytics and performance tracking
- **Sales Plugin**: Sales performance and pipeline analytics
- **Purchases Plugin**: Procurement analytics and vendor performance

## API Requirements

### Analytics Records Management

```http
GET    /api/analytics/records
POST   /api/analytics/records
GET    /api/analytics/records/{id}
PUT    /api/analytics/records/{id}
DELETE /api/analytics/records/{id}
```

### Dashboard Data

```http
GET    /api/analytics/dashboard/summary
GET    /api/analytics/dashboard/financial
GET    /api/analytics/dashboard/sales
GET    /api/analytics/dashboard/hr
GET    /api/analytics/reports/{type}
```

### Analytics Configuration

```http
GET    /api/analytics/dimensions
POST   /api/analytics/dimensions
GET    /api/analytics/kpis
POST   /api/analytics/kpis/calculate
```

## Usage Examples

### Creating Analytics Records

```php
// Cost center analytics
$costCenter = Record::create([
    'company_id' => 1,
    'name' => 'IT Department',
    'code' => 'DEPT_IT',
    'is_active' => true,
    'sort' => 10
]);

// Project analytics
$project = Record::create([
    'company_id' => 1,
    'name' => 'ERP Implementation',
    'code' => 'PROJ_ERP_2025',
    'is_active' => true,
    'sort' => 20
]);
```

### Integration with Other Modules

```php
// In account moves
$move->analytic_distribution = [
    'DEPT_IT' => 60.0,
    'PROJ_ERP_2025' => 40.0
];

// In employee records
$employee->analytic_record_code = 'DEPT_IT';

// In sales orders
$order->analytic_record_code = 'PROJ_ERP_2025';
```

This Analytics plugin provides the foundation for business intelligence across all modules in the ERP system.