# Remaining Plugins Documentation

This document covers the remaining plugins in the CIUIS ERP system that have simpler implementations or are placeholder/stub plugins.

## Table Views Plugin

### Overview
The Table Views plugin provides customizable data presentation framework for Filament tables across all modules.

### Features
- **Custom table configurations**: Save and load table view preferences
- **Column management**: Show/hide columns per user
- **Filter presets**: Save commonly used filter combinations
- **Sorting preferences**: Remember user sorting preferences

### Implementation
- Configuration-based (no database tables)
- Uses Filament's built-in table customization features
- User preferences stored in session/cookies

---

## Contacts Plugin

### Overview
Extended contact management system that builds on the Partners plugin foundation.

### Purpose
- **Enhanced CRM**: Additional contact management features
- **Relationship tracking**: Advanced partner relationship management
- **Communication logs**: Extended communication history
- **Contact categories**: Specialized contact classifications

### Integration
- Extends Partners plugin functionality
- Adds specialized contact types and relationships
- Enhanced search and filtering capabilities

---

## Invoices Plugin

### Overview
Dedicated invoice generation and management system that integrates with Accounts plugin.

### Core Features
- **Invoice templates**: Customizable invoice layouts
- **Recurring invoices**: Automated recurring billing
- **Payment tracking**: Invoice payment status management
- **Customer portal**: Customer access to invoices

### Database Schema
```sql
-- Main invoices table extending account moves
invoices_invoices (
    id, move_id, template_id, recurring_settings,
    customer_portal_access, payment_link, etc.
)

-- Invoice templates
invoices_templates (
    id, name, layout, fields, styling, etc.
)

-- Recurring invoice configurations
invoices_recurring_configs (
    id, invoice_id, frequency, next_date, etc.
)
```

---

## Payments Plugin

### Overview
Advanced payment processing system that extends the Accounts plugin payment functionality.

### Features
- **Payment gateways**: Integration with payment processors
- **Payment methods**: Extended payment method configurations
- **Payment tracking**: Advanced payment status tracking
- **Reconciliation**: Bank statement reconciliation features

### Integration Points
- Extends Accounts plugin payment system
- Integrates with bank statement processing
- Supports multiple payment gateways

---

## Projects Plugin

### Overview
Comprehensive project management system for tracking projects, tasks, and resources.

### Core Features
- **Project lifecycle**: From initiation to completion
- **Task management**: Hierarchical task breakdown
- **Resource allocation**: Team and equipment assignment
- **Time tracking**: Project time and cost tracking
- **Client management**: Integration with Partners plugin

### Database Schema
```sql
-- Projects
projects_projects (
    id, company_id, partner_id, name, description,
    start_date, end_date, status, budget, etc.
)

-- Project tasks
projects_tasks (
    id, project_id, parent_id, name, description,
    assigned_to, due_date, status, progress, etc.
)

-- Project resources
projects_resources (
    id, project_id, resource_type, resource_id,
    allocation_percentage, start_date, end_date
)

-- Time entries
projects_time_entries (
    id, project_id, task_id, user_id, hours,
    description, date, billable, etc.
)
```

---

## Recruitments Plugin

### Overview
Applicant tracking system for managing hiring processes and candidate evaluation.

### Features
- **Job postings**: Create and manage job openings
- **Application tracking**: Track candidate applications
- **Interview scheduling**: Manage interview processes
- **Candidate evaluation**: Scoring and feedback systems
- **Onboarding**: New hire onboarding workflows

### Database Schema
```sql
-- Job postings
recruitments_jobs (
    id, company_id, department_id, title,
    description, requirements, status, etc.
)

-- Applications
recruitments_applications (
    id, job_id, candidate_name, email,
    resume, status, applied_at, etc.
)

-- Interview stages
recruitments_interviews (
    id, application_id, stage, interviewer_id,
    scheduled_at, feedback, score, etc.
)
```

---

## Time-off Plugin

### Overview
Employee leave management system for tracking vacation, sick leave, and other time-off requests.

### Features
- **Leave types**: Configure different types of leave
- **Leave requests**: Employee leave request workflow
- **Approval process**: Manager approval workflows
- **Leave balances**: Track accrual and usage
- **Calendar integration**: Leave calendar visualization

### Database Schema
```sql
-- Leave types
timeoff_leave_types (
    id, company_id, name, accrual_rate,
    max_balance, requires_approval, etc.
)

-- Leave requests
timeoff_requests (
    id, employee_id, leave_type_id, start_date,
    end_date, days_requested, status, etc.
)

-- Leave balances
timeoff_balances (
    id, employee_id, leave_type_id, balance,
    accrued, used, year, etc.
)
```

---

## Timesheets Plugin

### Overview
Work hour tracking system for monitoring employee time and project allocation.

### Features
- **Time tracking**: Clock in/out functionality
- **Project time**: Time allocation to projects
- **Timesheet approval**: Manager approval process
- **Overtime tracking**: Track overtime hours
- **Reporting**: Time analysis and reporting

### Database Schema
```sql
-- Timesheets
timesheets_timesheets (
    id, employee_id, date, regular_hours,
    overtime_hours, status, approved_by, etc.
)

-- Time entries
timesheets_entries (
    id, timesheet_id, project_id, task_id,
    start_time, end_time, hours, description
)
```

---

## Website Plugin

### Overview
Customer-facing website functionality for e-commerce and customer portal features.

### Features
- **Public website**: Customer-facing website content
- **Product catalog**: Public product displays
- **Customer portal**: Customer account management
- **Order tracking**: Customer order status
- **Content management**: Website content administration

### Database Schema
```sql
-- Website pages
website_pages (
    id, company_id, title, slug, content,
    template, is_published, meta_data, etc.
)

-- Website menus
website_menus (
    id, company_id, name, location,
    menu_items, is_active, etc.
)

-- Customer portal access
website_customer_access (
    id, partner_id, access_token,
    permissions, last_login, etc.
)
```

---

## API Requirements Summary

### Common API Patterns

All remaining plugins follow similar REST API patterns:

```http
# Standard CRUD operations
GET    /api/{plugin}/{resource}
POST   /api/{plugin}/{resource}
GET    /api/{plugin}/{resource}/{id}
PUT    /api/{plugin}/{resource}/{id}
DELETE /api/{plugin}/{resource}/{id}

# Workflow actions
PUT    /api/{plugin}/{resource}/{id}/approve
PUT    /api/{plugin}/{resource}/{id}/reject
PUT    /api/{plugin}/{resource}/{id}/complete

# Reporting
GET    /api/{plugin}/reports/{type}
GET    /api/{plugin}/analytics/{metric}
```

### Integration Requirements

- **Authentication**: All APIs require JWT authentication
- **Authorization**: Role-based access control via permissions
- **Multi-company**: All resources are company-scoped
- **Real-time**: WebSocket support for live updates
- **File uploads**: Support for document attachments
- **Export**: PDF and Excel export capabilities

## Development Priority

### High Priority (Essential for ERP)
1. **Projects** - Project management is core ERP functionality
2. **Invoices** - Essential for billing and financial operations
3. **Payments** - Critical for financial processing

### Medium Priority (Enhanced Features)
1. **Time-off** - Important for HR management
2. **Timesheets** - Project time tracking
3. **Recruitments** - HR recruitment workflows

### Low Priority (Extended Features)
1. **Website** - Customer portal features
2. **Table Views** - UI customization
3. **Contacts** - Enhanced CRM features

## Migration Strategy for ShadCN/React

When building the new frontend:

1. **Start with high-priority plugins** that have substantial business logic
2. **Use the documented APIs** from the main plugins as patterns
3. **Implement core CRUD operations** first, then add advanced features
4. **Focus on user workflows** rather than admin configuration initially
5. **Add real-time features** after basic functionality is stable

This completes the documentation for all 22 plugins in the CIUIS ERP system, providing comprehensive coverage for building the new ShadCN/React frontend.