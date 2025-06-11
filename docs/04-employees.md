# Employees Plugin (HR Management) - Technical Documentation

## Overview

The Employees plugin (`webkul/employees`) provides comprehensive HR management functionality within the CiuisERP system. This plugin manages employee lifecycle, organizational structure, skills tracking, attendance management, and seamlessly integrates with the Partners plugin for unified contact management.

## Core Features

- **Employee Management**: Complete employee profiles with personal, work, and emergency information
- **Hierarchical Department Structure**: Multi-level department organization with manager assignments
- **Skills & Competency Management**: Skills tracking with levels and types
- **Job Positions**: Role definitions with skill requirements and recruitment planning
- **Calendar & Attendance**: Work schedule management with flexible time tracking
- **Resume & Career History**: Employee resume management with timeline display
- **Activity Planning Integration**: Department-based activity planning
- **Partner Integration**: Automatic partner creation for employees

## Database Schema

### Core Employee Tables

#### `employees_employees`
Central employee table with comprehensive personal and work information:

**Key Fields:**
- **Identity**: `id`, `name`, `employee_type`, `barcode`, `pin`
- **Work Info**: `job_title`, `work_email`, `work_phone`, `mobile_phone`
- **Personal**: `gender`, `birthday`, `marital`, `private_email`, `private_phone`
- **Address**: `private_street1/2`, `private_city`, `private_zip`, `private_state_id`, `private_country_id`
- **Family**: `spouse_complete_name`, `spouse_birthdate`, `children`, `emergency_contact`
- **Documentation**: `identification_id`, `passport_id`, `visa_no`, `permit_no`, `ssnid`, `sinid`
- **Work Status**: `is_active`, `departure_date`, `departure_reason_id`

**Key Relationships:**
- `company_id` → companies
- `user_id` → users (system user account)
- `partner_id` → partners_partners (auto-created partner record)
- `department_id` → employees_departments
- `job_id` → employees_job_positions
- `calendar_id` → employees_calendars
- `parent_id` → employees_employees (manager/supervisor)
- `coach_id` → employees_employees (mentor/coach)
- `bank_account_id` → partners_bank_accounts

#### `employees_departments`
Hierarchical department structure:

**Key Fields:**
- `name`, `complete_name` (full hierarchy path)
- `parent_id` → self-reference for hierarchy
- `manager_id` → employees_employees
- `master_department_id` → top-level department
- `parent_path` → hierarchical path for efficient queries
- `color` → visual identification

**Features:**
- Recursive hierarchy validation to prevent circular references
- Automatic complete name generation (`Parent / Child / Grandchild`)
- Path-based hierarchy tracking for efficient queries

### Skills Management

#### `employees_skill_types`
Categories of skills (e.g., Technical, Soft Skills, Languages):
- `name`, `sort` ordering

#### `employees_skills`
Individual skills within types:
- `name`, `skill_type_id`, `sort`

#### `employees_skill_levels`
Proficiency levels (Beginner, Intermediate, Expert):
- `name`, `sort`

#### `employees_employee_skills`
Employee skill assignments:
- `employee_id`, `skill_id`, `skill_level_id`, `skill_type_id`

### Job Management

#### `employees_job_positions`
Job role definitions:
- `name`, `description`, `requirements`
- `department_id`, `employment_type_id`
- `expected_employees`, `no_of_employee`, `no_of_recruitment`

#### `job_position_skills`
Skills required for job positions:
- Many-to-many relationship between job positions and skills

#### `employees_employment_types`
Employment categories (Full-time, Part-time, Contract):
- `name`, classification

### Calendar & Attendance

#### `employees_calendars`
Work schedule templates:
- `name`, `timezone`, `hours_per_day`
- `flexible_hours`, `two_weeks_calendar`
- `full_time_required_hours`

#### `employees_calendar_attendances`
Work schedule definitions:
- `calendar_id`, `day_of_week`, `day_period`
- `hour_from`, `hour_to`
- `week_type`, `display_type`
- Date ranges for temporary schedules

#### `employees_calendar_leaves`
Leave/absence definitions within calendars

### Employee Resume & History

#### `employees_employee_resume_line_types`
Resume entry categories (Education, Experience, Certifications)

#### `employees_employee_resumes`
Employee resume entries:
- `employee_id`, `employee_resume_line_type_id`
- `display_type`, `start_date`, `end_date`
- `name`, `description`

### Support Tables

#### `employees_work_locations`
Work location definitions:
- `name`, `location_type`, `location_number`
- Geographic and virtual work locations

#### `employees_categories`
Employee grouping categories

#### `employees_departure_reasons`
Standardized departure/termination reasons

## Business Logic & Model Relationships

### Employee Model (`Employee.php`)

**Key Traits:**
- `HasChatter`: Communication tracking
- `HasCustomFields`: Extensible field support
- `HasLogActivity`: Audit trail
- `SoftDeletes`: Safe deletion

**Critical Business Logic:**

#### Partner Integration
Automatic partner creation/update on employee save:

```php
protected static function boot()
{
    parent::boot();
    
    static::saved(function (self $employee) {
        if (! $employee->partner_id) {
            $employee->handlePartnerCreation($employee);
        } else {
            $employee->handlePartnerUpdation($employee);
        }
    });
}
```

**Partner Creation Logic:**
- Creates partner record with `account_type: 'individual'`, `sub_type: 'employee'`
- Maps employee data to partner fields (name, email, phone, etc.)
- Links employee and partner records bidirectionally

**Key Relationships:**
- Self-referencing hierarchy: `parent()`, `coach()`
- Work context: `department()`, `job()`, `workLocation()`, `calendar()`
- Personal info: `privateCountry()`, `privateState()`, `countryOfBirth()`
- Collections: `skills()`, `resumes()`, `categories()`

### Department Model (`Department.php`)

**Hierarchical Features:**
- Circular reference prevention
- Automatic path calculation
- Complete name generation with full hierarchy

**Key Methods:**
- `validateNoRecursion()`: Prevents circular department hierarchies
- `handleDepartmentData()`: Calculates paths and complete names
- `findTopLevelParentId()`: Identifies root department
- `getCompleteName()`: Builds full hierarchical name

### Activity Planning Integration

The plugin extends the base support system's activity planning:

```php
// ActivityPlan Model extends BaseActivityPlan
class ActivityPlan extends BaseActivityPlan
{
    public function department(): BelongsTo
    {
        return $this->belongsTo(Department::class);
    }
}
```

**Database Extension:**
- Adds `department_id` to `activity_plans` table
- Links activities to specific departments for better organization

## Enums & Configuration

### Core Enums

#### `MaritalStatus`
- Single, Married, Divorced, Widowed

#### `Gender`
- Male, Female, Other

#### `WorkLocation`
- Home, Office, Other (with Filament UI enhancements)

#### `DistanceUnit`
- Km, Miles for commute tracking

#### Calendar Enums
- `DayOfWeek`: Monday through Sunday
- `DayPeriod`: Morning, Afternoon
- `WeekType`: Odd, Even for two-week rotations
- `CalendarDisplayType`: Various display formats

## Integration Points

### Partners Plugin Integration

**Automatic Partner Creation:**
- Every employee automatically gets a corresponding partner record
- Partner data synced on employee updates
- Enables unified contact management across sales, purchases, accounting

**Partner Fields Mapping:**
```php
'account_type' => 'individual'
'sub_type' => 'employee'
'name' => employee.name
'email' => employee.work_email ?? employee.private_email
'phone' => employee.work_phone
'mobile' => employee.mobile_phone
'job_title' => employee.job_title
```

### Support Plugin Integration

**Activity Planning:**
- Departments can have associated activity plans
- Enables project management and task assignment by department

**Custom Fields:**
- All major entities support custom fields via `HasCustomFields` trait
- Extensible data model for organization-specific requirements

### Chatter Plugin Integration

**Communication Tracking:**
- Employees and departments support the chatter system
- Internal communication and note-taking capabilities
- Activity logging for audit trails

### Accounts Plugin Integration

**Bank Account Linking:**
- Employees can be linked to bank accounts via `bank_account_id`
- Supports payroll and expense management

## Filament Admin Interface

### Resource Structure

#### Employee Resource
- **Main Form**: Personal, work, and contact information
- **Skills Management**: Assign skills with proficiency levels
- **Resume Management**: Career history tracking
- **Custom Pages**: Specialized skill and resume management interfaces

#### Department Resource
- **Hierarchical Display**: Tree-view of department structure
- **Manager Assignment**: Link employees as department managers
- **Employee Listing**: View all employees in a department

#### Configuration Resources (Clustered)
- **Calendars**: Work schedule templates
- **Job Positions**: Role definitions with skill requirements
- **Work Locations**: Office and remote work locations
- **Skills Framework**: Skill types, skills, and proficiency levels
- **Employment Types**: Contract types and categories

### UI Features

**Global Search Integration:**
- Employees searchable by name, department, email, phone
- Quick access to employee records from global search

**Navigation Structure:**
- Primary navigation: Employees, Departments
- Configuration cluster: All supporting configuration
- Reporting cluster: Skills and performance reporting

## API Requirements & Extensibility

### Recommended API Endpoints

#### Employee Management
```
GET    /api/employees              # List employees with filtering
POST   /api/employees              # Create employee
GET    /api/employees/{id}         # Get employee details
PUT    /api/employees/{id}         # Update employee
DELETE /api/employees/{id}         # Soft delete employee

GET    /api/employees/{id}/skills  # Get employee skills
POST   /api/employees/{id}/skills  # Assign skill to employee
DELETE /api/employees/{id}/skills/{skill_id} # Remove skill

GET    /api/employees/{id}/resume  # Get employee resume
POST   /api/employees/{id}/resume  # Add resume entry
```

#### Department Management
```
GET    /api/departments            # List departments (tree structure)
POST   /api/departments            # Create department
GET    /api/departments/{id}       # Get department details
PUT    /api/departments/{id}       # Update department
DELETE /api/departments/{id}       # Delete department

GET    /api/departments/{id}/employees # Get department employees
GET    /api/departments/hierarchy       # Get full hierarchy tree
```

#### Skills & Job Management
```
GET    /api/skills                 # List all skills by type
GET    /api/job-positions          # List job positions
GET    /api/job-positions/{id}/skills # Get required skills for position
```

#### Calendar & Attendance
```
GET    /api/calendars              # List work calendars
GET    /api/calendars/{id}/schedule # Get calendar schedule
POST   /api/attendance             # Record attendance
GET    /api/employees/{id}/attendance # Get employee attendance
```

### Data Security Considerations

**Sensitive Information:**
- Personal data (SSN, passport, visa numbers)
- Financial data (bank account information)
- Emergency contacts and family information

**Access Control Requirements:**
- Role-based access to different employee information levels
- Department-based data access restrictions
- Audit logging for sensitive data access

**Privacy Compliance:**
- GDPR/privacy regulation compliance features
- Data retention policies
- Employee data export/deletion capabilities

## Performance Considerations

### Database Optimization

**Indexing Strategy:**
- Department hierarchy queries: Index on `parent_path`
- Employee searches: Composite indexes on `name`, `work_email`, `department_id`
- Skills queries: Index on `employee_id`, `skill_type_id`

**Query Optimization:**
- Department hierarchy: Path-based queries avoid recursive lookups
- Soft deletes: Include `deleted_at` in relevant indexes
- Partner integration: Optimize join queries for employee-partner data

### Caching Strategies

**Recommended Caching:**
- Department hierarchy trees (infrequently changing)
- Skills taxonomies (relatively static)
- Calendar schedules (per employee/department)
- Employee counts per department

## Migration & Deployment

### Initial Setup

1. **Run Migrations**: 19 migration files in chronological order
2. **Seed Data**: Run database seeder for basic setup
3. **Configure Permissions**: Set up role-based access
4. **Custom Fields**: Configure organization-specific fields

### Data Migration Considerations

**From External HR Systems:**
- Employee master data mapping
- Department hierarchy reconstruction
- Skills data normalization
- Historical resume/career data

**Partner Integration:**
- Existing partner records may need employee linking
- Duplicate detection and resolution
- Data synchronization strategies

## Extension Points

### Custom Field Integration

The plugin extensively uses the `HasCustomFields` trait, allowing:
- Organization-specific employee fields
- Department-specific metadata
- Custom skill categories and attributes
- Extended job position requirements

### Event Hooks

**Employee Lifecycle Events:**
- Employee creation/update triggers partner sync
- Department changes trigger hierarchy recalculation
- Skill assignments can trigger training recommendations

### Plugin Integrations

**Potential Extensions:**
- **Payroll Integration**: Link to accounting for salary management
- **Training Management**: Skills-based training recommendations
- **Performance Management**: Goal setting and evaluation
- **Recruitment Integration**: Job position to candidate tracking
- **Time Tracking**: Calendar integration with actual time logging

## Conclusion

The Employees plugin provides a comprehensive HR management foundation with:
- Complete employee lifecycle management
- Hierarchical organizational structure
- Skills and competency tracking
- Flexible calendar and attendance systems
- Seamless integration with other ERP modules

The plugin's architecture supports extensive customization and integration while maintaining data integrity and providing audit capabilities essential for HR management systems.