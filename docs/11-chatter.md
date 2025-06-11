# Chatter Plugin Documentation

## Overview

The Chatter plugin is a comprehensive internal communication system for the CIUIS ERP platform that provides polymorphic messaging capabilities, follower management, file attachments, and activity tracking for any entity in the system.

## Database Schema

### Core Tables

#### `chatter_followers`
**Purpose**: Manages polymorphic follower relationships
```sql
- id (Primary Key)
- followable_type (VARCHAR) - Polymorphic entity type
- followable_id (BIGINT) - Polymorphic entity ID  
- partner_id (FK to partners_partners) - Follower
- followed_at (TIMESTAMP) - When following started
- created_at, updated_at

UNIQUE KEY unique_follower (followable_type, followable_id, partner_id)
```

#### `chatter_messages`
**Purpose**: Stores all messages and activities in the system
```sql
- id (Primary Key)
- messageable_type (VARCHAR) - Polymorphic parent entity type
- messageable_id (BIGINT) - Polymorphic parent entity ID
- causer_type (VARCHAR) - Who created the message (polymorphic)
- causer_id (BIGINT) - Creator ID
- company_id (FK to companies)
- activity_type_id (FK to activity_types)
- assigned_to (FK to users) - User assignment
- type (ENUM: comment, activity, notification) - Message type
- subject (VARCHAR) - Optional subject line
- body (TEXT) - Rich text content
- summary (TEXT) - Brief description
- is_internal (BOOLEAN) - Internal vs external flag
- is_read (BOOLEAN) - Read status
- date_deadline (DATE) - Due date for activities
- pinned_at (TIMESTAMP) - Pinning timestamp
- properties (JSON) - Additional metadata
- created_at, updated_at
```

#### `chatter_attachments`
**Purpose**: File attachments for messages
```sql
- id (Primary Key)
- messageable_type (VARCHAR) - Polymorphic parent entity type
- messageable_id (BIGINT) - Polymorphic parent entity ID
- message_id (FK to chatter_messages)
- creator_id (FK to users) - File uploader
- company_id (FK to companies)
- file_path (VARCHAR) - Storage path
- original_file_name (VARCHAR) - Original filename
- mime_type (VARCHAR) - File MIME type
- file_size (INTEGER) - File size in bytes
- created_at, updated_at
```

## Business Logic & Models

### Core Models

#### Message Model
**Key Features**:
- Polymorphic relationships to parent entities
- Auto-sets causer on create/update
- JSON properties handling
- Activity management

**Relationships**:
```php
messageable() // Polymorphic to parent entity
causer() // Polymorphic to creator
company() // Belongs to company
activityType() // Belongs to activity type
assignedTo() // Belongs to user
attachments() // Has many attachments
```

#### Follower Model
**Key Features**:
- Polymorphic following system
- Tracks following timestamps
- Partner-based following

#### Attachment Model
**Key Features**:
- Auto-deletes files when model deleted
- URL accessor for file access
- File metadata management

## Entity Relationships & Integration

### HasChatter Trait

Any entity can use the chatter system by implementing the `HasChatter` trait:

#### Message Management
```php
// Core relationships
messages() // Get all messages (excludes activities)
activities() // Get activity-type messages
attachments() // Get all attachments

// Message operations
addMessage(array $data) // Add new message
replyToMessage(Message $parent, array $data) // Reply to existing message
removeMessage($id, $type = 'messages') // Delete message
pinMessage(Message $message) // Pin message
unpinMessage(Message $message) // Unpin message
```

#### Attachment Management
```php
// Attachment operations
addAttachments(array $files, array $data = []) // Add multiple files
removeAttachment($id) // Delete attachment
getAttachmentsByType(string $mimeType) // Filter by MIME type
getImageAttachments() // Get images only
getDocumentAttachments() // Get non-image files
```

#### Follower Management
```php
// Follower operations
followers() // Get all followers
addFollower(Partner $partner) // Add new follower
removeFollower(Partner $partner) // Remove follower
isFollowedBy(Partner $partner) // Check following status
```

### HasLogActivity Trait

Automatically logs model changes as activities:

#### Features
- **Auto-tracking**: Logs created, updated, deleted, restored events
- **Configurable attributes**: Define which fields to track
- **Relationship tracking**: Track changes in related models
- **Format handling**: Smart formatting for enums, booleans, JSON
- **Change detection**: Only logs actual changes

#### Usage
```php
class YourModel extends Model
{
    use HasChatter, HasLogActivity;
    
    protected $logAttributes = [
        'name',              // Direct attribute
        'status',           // Enum with getLabel() method
        'partner.name',     // Related model attribute
        'is_active'         // Boolean (converted to Yes/No)
    ];
}
```

## API Requirements

### Livewire Component Integration

The `ChatterPanel` provides real-time interface:

#### Core Actions
- **MessageAction**: Send messages with attachments
- **ActivityAction**: Create and manage activities
- **FollowerAction**: Manage followers
- **LogAction**: View activity logs
- **FileAction**: Handle file uploads

#### Features
- Real-time message display
- Activity management with due dates
- File attachment handling
- Follower notifications
- Message filtering and search

### Filament Integration

#### ChatterAction
Slide-over modal interface for Filament resources:
```php
use Webkul\Chatter\Filament\Actions\ChatterAction;

ChatterAction::make()
    ->setResource(YourResource::class)
    ->setActivityPlans($activityPlans)
    ->setMessageMailView('custom.mail.template')
```

#### ChatterWidget
Footer widget for record pages:
```php
use Webkul\Chatter\Filament\Widgets\ChatterWidget;

ChatterWidget::make()
    ->setRecord($record)
    ->setShowFollowers(true)
    ->setShowActivities(true)
```

### Message Types & Content

#### Supported Message Types
- `comment` - User-generated messages
- `activity` - Scheduled activities with due dates  
- `notification` - System-generated notifications
- `log` - Activity log entries

#### Content Structure
- **Subject**: Optional subject line
- **Body**: Rich text content with HTML support
- **Summary**: Brief description
- **Properties**: JSON metadata
- **Attachments**: Multiple file support

### File Attachment System

#### Supported File Types
- Images (all formats)
- PDF documents
- Microsoft Office documents (Word, Excel)
- Plain text files

#### Features
- File size validation (max 10MB)
- Automatic cleanup on deletion
- URL generation for access
- Preview capabilities

### Activity Management

#### Activity Features
- Due date tracking
- User assignment
- Activity type categorization
- Status tracking (pending, done, cancelled)
- Activity plan templates

### Email Notifications

#### MessageMail
- Supports file attachments
- Configurable templates
- Rich payload data

#### FollowerMail
- Follower-specific notifications
- Template customization

### Filtering & Search

#### Available Filters
- **Type**: Filter by message type
- **Internal**: Internal vs external messages
- **Date Range**: From/to date filtering
- **User**: Filter by creator or assignee
- **Activity Type**: Filter by activity type
- **Company**: Company-specific filtering
- **Search**: Text search across content

## Integration Examples

### Basic Usage
```php
use Webkul\Chatter\Traits\HasChatter;

class YourModel extends Model
{
    use HasChatter;
}

// Add message
$record->addMessage([
    'type' => 'comment',
    'body' => 'This is a message',
    'is_internal' => false
]);

// Add follower
$record->addFollower($partner);

// Add attachment
$record->addAttachments($files, [
    'type' => 'comment',
    'body' => 'Files attached'
]);
```

### Advanced Activity Management
```php
// Schedule activity
$record->addMessage([
    'type' => 'activity',
    'subject' => 'Follow up call',
    'body' => 'Call customer to discuss proposal',
    'assigned_to' => $userId,
    'date_deadline' => '2025-01-20',
    'activity_type_id' => $activityTypeId
]);

// Mark activity as done
$message->update(['properties' => ['status' => 'done']]);
```

This comprehensive communication system provides messaging, activity tracking, file management, and follower notifications that can be attached to any entity in the ERP system.