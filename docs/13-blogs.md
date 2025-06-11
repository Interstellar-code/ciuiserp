# Blogs Plugin Documentation

## Overview

The Blogs plugin provides a complete content management system for the CIUIS ERP platform, enabling organizations to manage internal and external blog content with categories, tags, and publishing workflows.

## Database Schema

### Core Tables

#### `blogs_categories`
```sql
CREATE TABLE blogs_categories (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    company_id BIGINT UNSIGNED NOT NULL,
    creator_id BIGINT UNSIGNED NOT NULL,
    parent_id BIGINT UNSIGNED NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT NULL,
    image VARCHAR(255) NULL,
    is_active BOOLEAN DEFAULT 1,
    sort INTEGER NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    deleted_at TIMESTAMP NULL,
    
    FOREIGN KEY (company_id) REFERENCES support_companies(id),
    FOREIGN KEY (creator_id) REFERENCES users(id),
    FOREIGN KEY (parent_id) REFERENCES blogs_categories(id),
    UNIQUE KEY unique_company_slug (company_id, slug),
    INDEX idx_parent (parent_id),
    INDEX idx_active (is_active)
);
```

#### `blogs_posts`
```sql
CREATE TABLE blogs_posts (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    company_id BIGINT UNSIGNED NOT NULL,
    creator_id BIGINT UNSIGNED NOT NULL,
    category_id BIGINT UNSIGNED NOT NULL,
    title VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    excerpt TEXT NULL,
    content LONGTEXT NOT NULL,
    featured_image VARCHAR(255) NULL,
    status ENUM('draft', 'published', 'archived') DEFAULT 'draft',
    is_featured BOOLEAN DEFAULT 0,
    published_at TIMESTAMP NULL,
    views_count INTEGER DEFAULT 0,
    likes_count INTEGER DEFAULT 0,
    meta_title VARCHAR(255) NULL,
    meta_description TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    deleted_at TIMESTAMP NULL,
    
    FOREIGN KEY (company_id) REFERENCES support_companies(id),
    FOREIGN KEY (creator_id) REFERENCES users(id),
    FOREIGN KEY (category_id) REFERENCES blogs_categories(id),
    UNIQUE KEY unique_company_slug (company_id, slug),
    INDEX idx_status (status),
    INDEX idx_published (published_at),
    INDEX idx_featured (is_featured)
);
```

#### `blogs_tags`
```sql
CREATE TABLE blogs_tags (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    company_id BIGINT UNSIGNED NOT NULL,
    creator_id BIGINT UNSIGNED NOT NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    color VARCHAR(7) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    deleted_at TIMESTAMP NULL,
    
    FOREIGN KEY (company_id) REFERENCES support_companies(id),
    FOREIGN KEY (creator_id) REFERENCES users(id),
    UNIQUE KEY unique_company_slug (company_id, slug)
);
```

#### `blogs_post_tags`
```sql
CREATE TABLE blogs_post_tags (
    post_id BIGINT UNSIGNED NOT NULL,
    tag_id BIGINT UNSIGNED NOT NULL,
    
    FOREIGN KEY (post_id) REFERENCES blogs_posts(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES blogs_tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);
```

## Business Logic

### Models

#### Category Model
- **Hierarchical structure**: Categories can have parent-child relationships
- **SEO-friendly slugs**: Automatic slug generation from names
- **Image support**: Featured images for categories
- **Sortable**: Custom ordering of categories
- **Soft deletes**: Safe deletion with recovery option

#### Post Model
- **Rich content**: Full WYSIWYG content with images and formatting
- **Publishing workflow**: Draft → Published → Archived states
- **SEO optimization**: Meta titles and descriptions
- **Engagement tracking**: Views and likes counters
- **Featured posts**: Highlight important content
- **Scheduled publishing**: Posts can be scheduled for future publication

#### Tag Model
- **Color coding**: Visual organization with color labels
- **Many-to-many**: Posts can have multiple tags
- **Company scoped**: Tags are company-specific

### Key Features

1. **Content Management**: Full WYSIWYG editor with media support
2. **SEO Optimization**: Meta tags, slugs, and structured data
3. **Publishing Workflow**: Draft, review, and publish processes
4. **Content Organization**: Categories and tags for classification
5. **Engagement Metrics**: Views, likes, and interaction tracking
6. **Multi-company Support**: Separate blog content per company

## Entity Relationships

### Post Relationships
```php
// Post belongs to category
category() -> Category

// Post belongs to creator
createdBy() -> User

// Post has many tags
tags() -> Tag[]

// Post belongs to company
company() -> Company
```

### Category Relationships
```php
// Category can have parent
parent() -> Category

// Category has many children
children() -> Category[]

// Category has many posts
posts() -> Post[]
```

## API Requirements

### Content Management

```http
GET    /api/blogs/posts
POST   /api/blogs/posts
GET    /api/blogs/posts/{slug}
PUT    /api/blogs/posts/{id}
DELETE /api/blogs/posts/{id}

GET    /api/blogs/categories
POST   /api/blogs/categories
GET    /api/blogs/categories/{slug}
PUT    /api/blogs/categories/{id}

GET    /api/blogs/tags
POST   /api/blogs/tags
PUT    /api/blogs/tags/{id}
```

### Publishing Workflow

```http
PUT    /api/blogs/posts/{id}/publish
PUT    /api/blogs/posts/{id}/unpublish
PUT    /api/blogs/posts/{id}/archive
POST   /api/blogs/posts/{id}/schedule
```

### Public API

```http
GET    /api/public/blogs/posts
GET    /api/public/blogs/posts/{slug}
GET    /api/public/blogs/categories
GET    /api/public/blogs/categories/{slug}/posts
POST   /api/public/blogs/posts/{id}/like
POST   /api/public/blogs/posts/{id}/view
```

### Analytics

```http
GET    /api/blogs/analytics/popular-posts
GET    /api/blogs/analytics/engagement-stats
GET    /api/blogs/analytics/category-performance
GET    /api/blogs/analytics/author-stats
```

## Usage Examples

### Creating Blog Content

```php
// Create category
$category = Category::create([
    'company_id' => 1,
    'name' => 'Company News',
    'slug' => 'company-news',
    'description' => 'Latest updates from our company',
    'is_active' => true
]);

// Create post
$post = Post::create([
    'company_id' => 1,
    'category_id' => $category->id,
    'title' => 'Welcome to Our New ERP System',
    'slug' => 'welcome-new-erp-system',
    'excerpt' => 'We are excited to announce...',
    'content' => '<p>Full article content...</p>',
    'status' => 'draft',
    'meta_title' => 'New ERP System Launch',
    'meta_description' => 'Learn about our new ERP system features'
]);

// Add tags
$post->tags()->attach([1, 2, 3]);

// Publish post
$post->update([
    'status' => 'published',
    'published_at' => now()
]);
```

### Content Organization

```php
// Hierarchical categories
$parent = Category::create(['name' => 'Technology']);
$child = Category::create([
    'name' => 'Software Updates',
    'parent_id' => $parent->id
]);

// Tagged content
$tag = Tag::create([
    'name' => 'ERP',
    'slug' => 'erp',
    'color' => '#3B82F6'
]);

$post->tags()->attach($tag);
```

This Blogs plugin provides comprehensive content management capabilities for both internal communication and external marketing purposes.