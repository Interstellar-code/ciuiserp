# Security Plugin Documentation

## Overview

The Security plugin is a core component of the CIUIS ERP system that provides comprehensive authentication, authorization, and access control functionality. Built on top of Laravel's authentication system and integrated with Filament Shield for role-based access control (RBAC), it manages user security across all plugins in the system.

## Architecture

### Core Components

1. **Authentication System** - Laravel-based user authentication
2. **Authorization Layer** - Spatie Permission package for roles and permissions
3. **Multi-Company Support** - Company-scoped access control
4. **Team Management** - Team-based grouping and permissions
5. **User Invitation System** - Secure user onboarding
6. **Scoped Permissions** - Individual, Group, and Global access levels

## Database Schema

### Primary Tables

#### `users` Table
Base user table extended with security-specific fields:

```sql
- id (Primary Key)
- name
- email  
- password
- partner_id (Foreign Key -> partners_partners.id)
- language
- is_active
- default_company_id (Foreign Key -> companies.id)
- resource_permission (ENUM: 'individual', 'group', 'global')
- created_at, updated_at
```

#### `user_invitations` Table
Manages user invitation workflow:

```sql
- id (Primary Key)
- email
- created_at, updated_at
```

#### `teams` Table
Team management for group-based permissions:

```sql
- id (Primary Key)
- name
- created_at, updated_at
```

#### `user_team` Table (Pivot)
Many-to-many relationship between users and teams:

```sql
- user_id (Foreign Key -> users.id)
- team_id (Foreign Key -> teams.id)
```

### Permission Tables (Spatie Package)

#### `permissions` Table
```sql
- id (Primary Key)
- name
- guard_name
- created_at, updated_at
```

#### `roles` Table
```sql
- id (Primary Key)
- name
- guard_name
- created_at, updated_at
```

#### `model_has_permissions` Table
```sql
- permission_id (Foreign Key)
- model_type
- model_id
```

#### `model_has_roles` Table
```sql
- role_id (Foreign Key)
- model_type
- model_id
```

#### `role_has_permissions` Table
```sql
- permission_id (Foreign Key)
- role_id (Foreign Key)
```

## Models and Relationships

### User Model (`Webkul\Security\Models\User`)

Extends the base Laravel User model with security features:

**Key Features:**
- Implements Filament panel access control
- Soft deletes support
- Spatie roles and permissions integration
- Automatic partner creation/synchronization

**Relationships:**
```php
// Team relationships
public function teams(): BelongsToMany

// Employee relationship
public function employee(): HasOne

// Department management
public function departments(): HasMany

// Company relationships
public function companies(): HasMany
public function defaultCompany(): BelongsTo
public function allowedCompanies(): BelongsToMany

// Partner integration
public function partner(): BelongsTo
```

**Key Methods:**
- `canAccessPanel(Panel $panel): bool` - Controls Filament panel access
- `getAvatarUrlAttribute()` - Retrieves user avatar from partner
- `handlePartnerCreation()` - Automatic partner record creation
- `handlePartnerUpdation()` - Partner record synchronization

### Team Model (`Webkul\Security\Models\Team`)

Simple team management model for group-based permissions:

**Relationships:**
```php
public function users(): BelongsToMany
```

### Invitation Model (`Webkul\Security\Models\Invitation`)

Manages user invitation process:

**Features:**
- Mass assignable fields
- Factory support for testing
- Integration with mail system

## Permission System

### Permission Types (Enums)

#### `PermissionType` Enum
Defines access control levels:

- **`INDIVIDUAL`** - Users can only access their own resources
- **`GROUP`** - Users can access resources of team members
- **`GLOBAL`** - Users can access all resources

#### `CompanyStatus` Enum
Company activation states:

- **`ACTIVE`** - Company is operational
- **`INACTIVE`** - Company is disabled

### Permission Scoping

#### `HasScopedPermissions` Trait

Provides methods for checking user access levels:

```php
// Check if user has global access
protected function hasGlobalAccess(User $user): bool

// Check if user has group access (team-based)
protected function hasGroupAccess(User $user, Model $model, string $ownerAttribute = 'user'): bool

// Check if user has individual access (own resources only)
protected function hasIndividualAccess(User $user, Model $model, $ownerAttribute = 'user'): bool

// Main access method combining all permission types
protected function hasAccess(User $user, Model $model, string $ownerAttribute = 'user'): bool
```

#### `UserPermissionScope` Global Scope

Automatically filters query results based on user permission level:

- **Individual Level**: Shows only user's own records + followed records
- **Group Level**: Shows records from users in same teams
- **Global Level**: Shows all records (no filtering)

## Authorization Policies

### UserPolicy (`Webkul\Security\Policies\UserPolicy`)

Standard CRUD permissions for user management:
- `viewAny` - List users
- `view` - View specific user
- `create` - Create new users
- `update` - Modify users
- `delete` - Delete users
- `deleteAny` - Bulk delete
- `forceDelete` - Permanent deletion
- `forceDeleteAny` - Bulk permanent deletion
- `restore` - Restore soft-deleted users
- `restoreAny` - Bulk restore
- `replicate` - Duplicate user records
- `reorder` - Change user ordering

### RolePolicy (`Webkul\Security\Policies\RolePolicy`)

Role management permissions:
- Similar structure to UserPolicy
- Controls access to role creation and management
- Note: Contains some inconsistencies referencing 'field' permissions (likely copy-paste errors)

### TeamPolicy (`Webkul\Security\Policies\TeamPolicy`)

Team management permissions following the same pattern.

## User Invitation System

### Workflow

1. **Invitation Creation**: Admin creates invitation with email address
2. **Email Dispatch**: System sends signed invitation URL via email
3. **Acceptance Process**: User clicks link and completes registration
4. **Account Activation**: User account is created and activated

### Components

#### `UserInvitationMail` Class
- Generates signed URLs for security
- Uses Markdown template for email content
- Supports localization

#### `AcceptInvitation` Livewire Component
- Handles invitation acceptance workflow
- Validates invitation tokens
- Creates user accounts

### Security Features
- **Signed URLs**: Prevents invitation tampering
- **Time-based Expiration**: Invitations can expire
- **Single-use Tokens**: Prevents invitation reuse

## Settings Management

### `UserSettings` Class

Global user-related configuration:

```php
public bool $enable_user_invitation;    // Control invitation system
public bool $enable_reset_password;     // Control password reset
public ?int $default_role_id;          // Default role for new users
public ?int $default_company_id;       // Default company assignment
```

## Integration Points

### Multi-Plugin Integration

The Security plugin integrates with other system components:

1. **Partner Plugin**: Automatic partner record creation for users
2. **Employee Plugin**: User-employee relationship management  
3. **Company Plugin**: Multi-company access control
4. **Chatter Plugin**: User following and notification system
5. **Field Plugin**: Custom field support for user records

### Filament Integration

#### Resources
- **UserResource**: Complete user management interface
- **RoleResource**: Role and permission management
- **TeamResource**: Team management interface
- **CompanyResource**: Company management with branch relationships

#### Features
- Global search functionality
- Bulk operations support
- Advanced filtering and sorting
- Responsive design
- Multi-language support

## API Requirements

### Authentication APIs

#### User Authentication
```php
// Login endpoint
POST /api/auth/login
{
    "email": "user@example.com",
    "password": "password"
}

// Response
{
    "access_token": "...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "user": {...}
}
```

#### Token Refresh
```php
POST /api/auth/refresh
Authorization: Bearer {token}
```

#### Logout
```php
POST /api/auth/logout
Authorization: Bearer {token}
```

### Role Management APIs

#### List Roles
```php
GET /api/roles
Authorization: Bearer {token}
```

#### Create Role
```php
POST /api/roles
Authorization: Bearer {token}
{
    "name": "Role Name",
    "permissions": ["permission1", "permission2"]
}
```

#### Update Role
```php
PUT /api/roles/{id}
Authorization: Bearer {token}
{
    "name": "Updated Role Name",
    "permissions": ["permission1", "permission3"]
}
```

### Permission Handling APIs

#### List Permissions
```php
GET /api/permissions
Authorization: Bearer {token}
```

#### Check User Permissions
```php
GET /api/user/permissions
Authorization: Bearer {token}
```

#### Assign Permissions
```php
POST /api/users/{id}/permissions
Authorization: Bearer {token}
{
    "permissions": ["permission1", "permission2"]
}
```

### Team Management APIs

#### List Teams
```php
GET /api/teams
Authorization: Bearer {token}
```

#### Create Team
```php
POST /api/teams
Authorization: Bearer {token}
{
    "name": "Team Name"
}
```

#### Add User to Team
```php
POST /api/teams/{id}/users
Authorization: Bearer {token}
{
    "user_ids": [1, 2, 3]
}
```

## Security Considerations

### Access Control
1. **Permission-based Authorization**: All actions require specific permissions
2. **Company Scoping**: Users can only access data from allowed companies
3. **Team-based Access**: Group permissions limit access to team resources
4. **Owner-based Access**: Individual permissions restrict to owned resources

### Data Protection
1. **Soft Deletes**: User data can be recovered if accidentally deleted
2. **Audit Trails**: Changes tracked through chatter system
3. **Signed URLs**: Secure invitation and password reset links
4. **Password Security**: Laravel's built-in password hashing

### Session Management
1. **Token-based Authentication**: API access through Bearer tokens
2. **Session Timeouts**: Configurable session expiration
3. **Multi-device Support**: Users can have multiple active sessions

## Configuration

### Key Configuration Files

#### `config/permission.php`
```php
'teams' => false,                    // Enable team support
'table_names' => [...],             // Permission table names
'column_names' => [...],            // Column name mappings
```

#### `config/filament-shield.php`
```php
'shield_resource' => [
    'slug' => 'shield/roles',
    'navigation_sort' => -1,
    'navigation_badge' => true,
],
```

### Environment Variables
```env
# Authentication
AUTH_PROVIDER=eloquent
AUTH_MODEL=Webkul\Security\Models\User

# Session Configuration
SESSION_LIFETIME=120
SESSION_ENCRYPT=false

# Password Reset
MAIL_FROM_ADDRESS=noreply@example.com
MAIL_FROM_NAME="${APP_NAME}"
```

## Best Practices

### User Management
1. Always assign appropriate roles to new users
2. Use teams for department-based access control
3. Regularly audit user permissions
4. Implement strong password policies

### Permission Design
1. Follow principle of least privilege
2. Create granular permissions for specific actions
3. Use role hierarchies effectively
4. Document permission requirements

### Security Maintenance
1. Regular security audits
2. Monitor failed authentication attempts
3. Update dependencies regularly
4. Implement proper logging and monitoring

## Troubleshooting

### Common Issues

#### Permission Denied Errors
1. Check user role assignments
2. Verify permission configurations
3. Clear permission cache: `php artisan permission:cache-reset`

#### Invitation System Issues
1. Verify mail configuration
2. Check signed URL expiration
3. Validate invitation table records

#### Multi-Company Access Problems
1. Verify user company assignments
2. Check default company configuration
3. Review company-specific permissions

### Debug Commands
```bash
# Clear permission cache
php artisan permission:cache-reset

# List all permissions
php artisan permission:show

# Create role with permissions
php artisan shield:generate

# Check user permissions
php artisan tinker
> User::find(1)->getAllPermissions()
```

## Migration Guide

### From Basic Laravel Auth
1. Install Spatie Permission package
2. Run permission migrations
3. Update User model to extend Security User
4. Configure Filament Shield
5. Assign initial roles and permissions

### Adding Multi-Company Support
1. Run company-related migrations
2. Update user relationships
3. Configure company scoping
4. Update policies for company access

## Testing

### Unit Tests
```php
// Test user permissions
public function test_user_can_access_own_resources()
{
    $user = User::factory()->create(['resource_permission' => 'individual']);
    $this->assertTrue($user->hasAccess($user->ownedModel));
}

// Test team permissions
public function test_user_can_access_team_resources()
{
    $user = User::factory()->create(['resource_permission' => 'group']);
    $team = Team::factory()->create();
    $user->teams()->attach($team);
    // ... test team access
}
```

### Integration Tests
```php
// Test invitation workflow
public function test_user_invitation_workflow()
{
    $invitation = Invitation::factory()->create();
    Mail::fake();
    
    // Send invitation
    Mail::to($invitation->email)->send(new UserInvitationMail($invitation));
    
    // Verify email sent
    Mail::assertSent(UserInvitationMail::class);
}
```

## Performance Considerations

### Optimization Strategies
1. **Permission Caching**: Enable Spatie permission caching
2. **Query Optimization**: Use eager loading for relationships
3. **Database Indexing**: Index foreign keys and frequently queried columns
4. **Session Storage**: Use Redis for session storage in production

### Monitoring
1. Track authentication failure rates
2. Monitor permission check performance
3. Watch for slow queries in permission tables
4. Set up alerts for security events

This comprehensive documentation covers all aspects of the Security plugin's architecture, functionality, and integration points within the CIUIS ERP system.