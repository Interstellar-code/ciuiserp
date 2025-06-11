# Fields Plugin Documentation

## Overview

The Fields plugin provides a comprehensive polymorphic custom field system for the CIUIS ERP application. It allows administrators to dynamically add custom fields to any entity in the system without modifying core code or database schemas.

## Core Features

- **Polymorphic Design**: Attach custom fields to any model that uses the `HasCustomFields` trait
- **Dynamic Schema Management**: Automatically adds columns to existing tables when fields are created
- **Filament Integration**: Seamless integration with Filament forms, tables, infolists, and filters
- **Validation Support**: Comprehensive validation rules for different field types
- **Sortable Fields**: Fields can be ordered and sorted within forms and tables
- **Soft Deletes**: Fields support soft deletion for data integrity

## Database Schema

### Custom Fields Table

```sql
CREATE TABLE custom_fields (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    code VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(255) NOT NULL,
    input_type VARCHAR(255) NULL,
    is_multiselect BOOLEAN DEFAULT 0,
    datalist JSON NULL,
    options JSON NULL,
    form_settings JSON NULL,
    use_in_table BOOLEAN DEFAULT 0,
    table_settings JSON NULL,
    infolist_settings JSON NULL,
    sort INTEGER NULL,
    customizable_type VARCHAR(255) NOT NULL,
    deleted_at TIMESTAMP NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    
    UNIQUE KEY unique_code_type (code, customizable_type),
    INDEX idx_code (code),
    INDEX idx_sort (sort)
);
```

### Key Schema Elements

- **code**: Unique identifier for the field within a model type
- **customizable_type**: The model class that this field applies to (polymorphic key)
- **type**: Field type (text, select, checkbox, etc.)
- **input_type**: Sub-type for text fields (email, numeric, etc.)
- **options**: JSON array of options for select/radio/checkbox_list fields
- **form_settings**: JSON configuration for form behavior and validation
- **table_settings**: JSON configuration for table display
- **infolist_settings**: JSON configuration for infolist display

## Business Logic

### Field Model

```php
class Field extends Model implements Sortable
{
    use SoftDeletes, SortableTrait;
    
    protected $table = 'custom_fields';
    
    protected $fillable = [
        'code', 'name', 'type', 'input_type', 'is_multiselect',
        'datalist', 'options', 'form_settings', 'use_in_table',
        'table_settings', 'infolist_settings', 'sort', 'customizable_type'
    ];
    
    protected $casts = [
        'is_multiselect' => 'boolean',
        'options' => 'array',
        'form_settings' => 'array',
        'table_settings' => 'array',
        'infolist_settings' => 'array',
    ];
}
```

### HasCustomFields Model Trait

This trait provides the core functionality for models to support custom fields:

```php
trait HasCustomFields
{
    protected static function bootHasCustomFields()
    {
        static::retrieved(fn ($model) => $model->loadCustomFields());
        static::creating(fn ($model) => $model->loadCustomFields());
        static::updating(fn ($model) => $model->loadCustomFields());
    }
    
    protected function loadCustomFields()
    {
        $customFields = $this->getCustomFields();
        $this->mergeFillable($customFields->pluck('code')->toArray());
        $this->mergeCasts($customFields->select('code', 'type', 'is_multiselect')->get());
    }
}
```

### Column Management

The `FieldsColumnManager` manages database schema changes when custom fields are created, updated, or deleted:

```php
class FieldsColumnManager
{
    public static function createColumn(Field $field): void
    public static function updateColumn(Field $field): void
    public static function deleteColumn(Field $field): void
    
    protected static function getColumnType(Field $field): string
    {
        return match ($field->type) {
            'text' => static::getTextColumnType($field),
            'textarea', 'editor', 'markdown' => 'text',
            'select' => $field->is_multiselect ? 'json' : 'string',
            'checkbox', 'toggle' => 'boolean',
            'checkbox_list' => 'json',
            'datetime' => 'datetime',
            'color' => 'string',
            default => 'string'
        };
    }
}
```

## Entity Relationships

### Polymorphic Relationship

The Fields plugin uses a polymorphic relationship through the `customizable_type` column:

```php
// Field belongs to any model type
Field::where('customizable_type', Employee::class)->get();

// Models using HasCustomFields trait automatically get custom fields
$employee = Employee::find(1);
$employee->custom_field_code; // Automatically available
```

### Supported Field Types

1. **Text Fields**
   - `text`: Basic text input with sub-types (email, numeric, integer, password, tel, url, color)
   - `textarea`: Multi-line text input
   - `editor`: Rich text editor
   - `markdown`: Markdown editor

2. **Selection Fields**
   - `select`: Dropdown selection (single or multiple)
   - `radio`: Radio button selection
   - `checkbox_list`: Multiple checkbox selection

3. **Boolean Fields**
   - `checkbox`: Single checkbox
   - `toggle`: Toggle switch

4. **Date/Time Fields**
   - `datetime`: Date and time picker

5. **Special Fields**
   - `color`: Color picker

## Usage Examples

### Adding Custom Fields to a Model

```php
// 1. Add the trait to your model
class Employee extends Model
{
    use HasCustomFields;
    // ... other model code
}

// 2. Add the trait to your Filament Resource
class EmployeeResource extends Resource
{
    use HasCustomFields;
    
    public static function form(Form $form): Form
    {
        return $form->schema([
            // Base form fields
            TextInput::make('name'),
            // Merge custom fields
            ...static::getCustomFormFields(),
        ]);
    }
}
```

### Creating Custom Fields via Admin Interface

1. Navigate to Settings > Custom Fields
2. Click "Create Field"
3. Configure field properties:
   - **Name**: Human-readable field name
   - **Code**: Database column name (must be unique per model)
   - **Type**: Field type (text, select, checkbox, etc.)
   - **Resource**: Target model class
   - **Options**: For select/radio/checkbox_list fields
   - **Form Settings**: Validation rules and display options
   - **Table Settings**: How field appears in data tables
   - **Infolist Settings**: How field appears in detail views

## API Requirements

### Field Management APIs

#### Create Field
```http
POST /admin/custom-fields
{
    "name": "Employee Level",
    "code": "employee_level",
    "type": "select",
    "customizable_type": "Webkul\\Employee\\Models\\Employee",
    "options": ["Junior", "Senior", "Lead", "Manager"],
    "form_settings": {
        "validations": [
            {"validation": "required"}
        ]
    },
    "use_in_table": true,
    "sort": 10
}
```

#### Update Field
```http
PUT /admin/custom-fields/{id}
{
    "name": "Updated Field Name",
    "form_settings": {
        "validations": [
            {"validation": "required"},
            {"validation": "in", "value": "Junior,Senior,Lead"}
        ]
    }
}
```

#### Delete Field
```http
DELETE /admin/custom-fields/{id}
```

### Field Configuration Examples

#### Text Field with Validation
```json
{
    "code": "employee_id",
    "name": "Employee ID",
    "type": "text",
    "input_type": "text",
    "form_settings": {
        "validations": [
            {"validation": "required"},
            {"validation": "unique"},
            {"validation": "regex", "value": "/^EMP[0-9]{4}$/"}
        ],
        "settings": [
            {"setting": "placeholder", "value": "EMP0000"},
            {"setting": "helperText", "value": "Format: EMP followed by 4 digits"}
        ]
    }
}
```

#### Multi-Select Field
```json
{
    "code": "skills",
    "name": "Skills",
    "type": "select",
    "is_multiselect": true,
    "options": ["PHP", "Laravel", "JavaScript", "Vue.js", "React"],
    "form_settings": {
        "validations": [
            {"validation": "minItems", "value": "1"},
            {"validation": "maxItems", "value": "5"}
        ]
    },
    "use_in_table": true,
    "table_settings": [
        {"setting": "limit", "value": "3"},
        {"setting": "separator", "value": ", "}
    ]
}
```

## Security & Permissions

The Fields plugin integrates with the security system through policies:

```php
class FieldPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->can('view_any_field');
    }
    
    public function create(User $user): bool
    {
        return $user->can('create_field');
    }
    
    public function update(User $user, Field $field): bool
    {
        return $user->can('update_field');
    }
    
    public function delete(User $user, Field $field): bool
    {
        return $user->can('delete_field');
    }
}
```

## Performance Considerations

1. **Column Addition**: Adding custom fields creates real database columns, maintaining query performance
2. **Caching**: Field definitions are cached during model lifecycle events
3. **Lazy Loading**: Custom fields are only loaded when the model is accessed
4. **Index Support**: Common field types automatically get appropriate database indexes

## Migration and Maintenance

### Adding Fields Programmatically

```php
use Webkul\Field\Models\Field;
use Webkul\Field\FieldsColumnManager;

$field = Field::create([
    'code' => 'custom_field',
    'name' => 'Custom Field',
    'type' => 'text',
    'customizable_type' => Employee::class,
    'sort' => 10,
]);

FieldsColumnManager::createColumn($field);
```

### Removing Fields

```php
$field = Field::find($id);
FieldsColumnManager::deleteColumn($field);
$field->forceDelete();
```

This comprehensive documentation covers the Fields plugin's architecture, implementation, and usage patterns within the CIUIS ERP system.