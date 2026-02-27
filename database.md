# WordPress Database - Complete Guide

> **Think of the WordPress database as a filing cabinet.** Each drawer (table) stores one type of information. The drawers are connected by labels (foreign keys) so you can find everything about a post, user, or comment by following the links.

---

## Default Tables Overview

A fresh WordPress install creates **12 tables** (prefix `wp_` by default, configurable in `wp-config.php`).

```
wp_posts              All content: posts, pages, attachments, revisions
wp_postmeta           Extra data attached to any post (key/value pairs)
wp_users              Registered user accounts
wp_usermeta           Extra data attached to any user (key/value pairs)
wp_terms              Tag/category/taxonomy term names
wp_term_taxonomy      Which taxonomy a term belongs to (category, tag, …)
wp_term_relationships Links posts to their terms (categories, tags)
wp_termmeta           Extra data attached to any term
wp_comments           Comments on posts
wp_commentmeta        Extra data attached to any comment
wp_options            Site-wide settings (siteurl, blogname, …)
wp_links              Blogroll links (legacy, rarely used)
```

---

## Table Details

### `wp_posts`

The core of WordPress — stores every piece of content.

| Column | Type | Description |
|---|---|---|
| `ID` | BIGINT UNSIGNED | Primary key, auto-increment |
| `post_author` | BIGINT UNSIGNED | FK → `wp_users.ID` |
| `post_date` | DATETIME | Publication date (site timezone) |
| `post_date_gmt` | DATETIME | Publication date (UTC) |
| `post_content` | LONGTEXT | Full body content |
| `post_title` | TEXT | Title |
| `post_excerpt` | TEXT | Short excerpt |
| `post_status` | VARCHAR(20) | `publish`, `draft`, `pending`, `trash`, `auto-draft`, `future`, `private`, `inherit` |
| `comment_status` | VARCHAR(20) | `open` or `closed` |
| `ping_status` | VARCHAR(20) | `open` or `closed` |
| `post_password` | VARCHAR(255) | Plain-text password for protected posts |
| `post_name` | VARCHAR(200) | URL slug (must be unique per status) |
| `to_ping` | TEXT | URLs to ping |
| `pinged` | TEXT | URLs already pinged |
| `post_modified` | DATETIME | Last modification date (site timezone) |
| `post_modified_gmt` | DATETIME | Last modification date (UTC) |
| `post_content_filtered` | LONGTEXT | Reserved for plugins |
| `post_parent` | BIGINT UNSIGNED | FK → `wp_posts.ID` (self-join for pages, attachments, revisions) |
| `guid` | VARCHAR(255) | Globally unique identifier (permanent link, never changes) |
| `menu_order` | INT | Sort order for pages and nav items |
| `post_type` | VARCHAR(20) | `post`, `page`, `attachment`, `revision`, `nav_menu_item`, or any custom type |
| `post_mime_type` | VARCHAR(100) | MIME type for attachments (e.g. `image/jpeg`) |
| `comment_count` | BIGINT | Cached count of approved comments |

**Sample data:**

```sql
SELECT ID, post_title, post_type, post_status, post_author
FROM wp_posts
LIMIT 5;
```

```
+----+----------------------------+-----------+-------------+--------------+
| ID | post_title                 | post_type | post_status | post_author  |
+----+----------------------------+-----------+-------------+--------------+
|  1 | Hello World!               | post      | publish     | 1            |
|  2 | Sample Page                | page      | publish     | 1            |
|  3 | Auto Draft                 | post      | auto-draft  | 1            |
|  4 | Privacy Policy             | page      | draft       | 1            |
|  5 | My Featured Image          | attachment| inherit     | 1            |
+----+----------------------------+-----------+-------------+--------------+
```

---

### `wp_postmeta`

Stores unlimited key/value pairs for any post (custom fields, plugin data, etc.).

| Column | Type | Description |
|---|---|---|
| `meta_id` | BIGINT UNSIGNED | Primary key |
| `post_id` | BIGINT UNSIGNED | FK → `wp_posts.ID` |
| `meta_key` | VARCHAR(255) | Key name (plugin convention: prefix with `_` for hidden fields) |
| `meta_value` | LONGTEXT | Serialised or plain value |

**Sample data:**

```sql
SELECT meta_id, post_id, meta_key, meta_value
FROM wp_postmeta
WHERE post_id = 1;
```

```
+---------+---------+---------------------------+---------------------------+
| meta_id | post_id | meta_key                  | meta_value                |
+---------+---------+---------------------------+---------------------------+
|       1 |       1 | _edit_lock                | 1700000000:1              |
|       2 |       1 | _thumbnail_id             | 5                         |
|       3 |       1 | _yoast_wpseo_metadesc     | My SEO description here   |
|       4 |       1 | _price                    | 49.99                     |
+---------+---------+---------------------------+---------------------------+
```

---

### `wp_users`

One row per registered user account.

| Column | Type | Description |
|---|---|---|
| `ID` | BIGINT UNSIGNED | Primary key |
| `user_login` | VARCHAR(60) | Username (unique) |
| `user_pass` | VARCHAR(255) | Hashed password (phpass) |
| `user_nicename` | VARCHAR(50) | URL-friendly display name (slug) |
| `user_email` | VARCHAR(100) | Email address (unique) |
| `user_url` | VARCHAR(100) | Website URL |
| `user_registered` | DATETIME | Registration date (UTC) |
| `user_activation_key` | VARCHAR(255) | Password-reset token |
| `user_status` | INT | Legacy field (always 0 in modern WP) |
| `display_name` | VARCHAR(250) | Public display name |

**Sample data:**

```sql
SELECT ID, user_login, user_email, display_name, user_registered
FROM wp_users;
```

```
+----+------------+----------------------+--------------+---------------------+
| ID | user_login | user_email           | display_name | user_registered     |
+----+------------+----------------------+--------------+---------------------+
|  1 | admin      | admin@example.com    | Admin        | 2024-01-01 10:00:00 |
|  2 | jane_doe   | jane@example.com     | Jane Doe     | 2024-03-15 08:30:00 |
|  3 | john_smith | john@example.com     | John Smith   | 2024-06-20 14:00:00 |
+----+------------+----------------------+--------------+---------------------+
```

---

### `wp_usermeta`

Stores unlimited key/value pairs for any user (roles, preferences, plugin data).

| Column | Type | Description |
|---|---|---|
| `umeta_id` | BIGINT UNSIGNED | Primary key |
| `user_id` | BIGINT UNSIGNED | FK → `wp_users.ID` |
| `meta_key` | VARCHAR(255) | Key name |
| `meta_value` | LONGTEXT | Value (roles stored as serialised array) |

**Sample data:**

```sql
SELECT umeta_id, user_id, meta_key, meta_value
FROM wp_usermeta
WHERE user_id = 1;
```

```
+----------+---------+------------------------------+-------------------------------------+
| umeta_id | user_id | meta_key                     | meta_value                          |
+----------+---------+------------------------------+-------------------------------------+
|        1 |       1 | wp_capabilities              | a:1:{s:13:"administrator";b:1;}     |
|        2 |       1 | wp_user_level                | 10                                  |
|        3 |       1 | show_welcome_panel           | 1                                   |
|        4 |       1 | session_tokens               | a:1:{s:64:"abc...";}                |
+----------+---------+------------------------------+-------------------------------------+
```

> **Note:** The `wp_capabilities` key uses the table prefix so it changes with `$table_prefix` in `wp-config.php` (e.g. `mywp_capabilities`).

---

### `wp_terms`

Stores the actual term names — categories, tags, or any custom taxonomy term.

| Column | Type | Description |
|---|---|---|
| `term_id` | BIGINT UNSIGNED | Primary key |
| `name` | VARCHAR(200) | Display name (e.g. "Technology") |
| `slug` | VARCHAR(200) | URL-friendly slug (e.g. "technology") |
| `term_group` | BIGINT | Reserved for grouping aliases (rarely used) |

**Sample data:**

```sql
SELECT term_id, name, slug FROM wp_terms LIMIT 5;
```

```
+---------+---------------+---------------+
| term_id | name          | slug          |
+---------+---------------+---------------+
|       1 | Uncategorized | uncategorized |
|       2 | Technology    | technology    |
|       3 | WordPress     | wordpress     |
|       4 | Tutorial      | tutorial      |
|       5 | News          | news          |
+---------+---------------+---------------+
```

---

### `wp_term_taxonomy`

Pairs a term with its taxonomy, and tracks the parent/count.

| Column | Type | Description |
|---|---|---|
| `term_taxonomy_id` | BIGINT UNSIGNED | Primary key |
| `term_id` | BIGINT UNSIGNED | FK → `wp_terms.term_id` |
| `taxonomy` | VARCHAR(32) | `category`, `post_tag`, `nav_menu`, `link_category`, or custom |
| `description` | LONGTEXT | Optional description |
| `parent` | BIGINT UNSIGNED | FK → `wp_term_taxonomy.term_taxonomy_id` (for hierarchical terms) |
| `count` | BIGINT | Cached count of objects using this term |

**Sample data:**

```sql
SELECT tt.term_taxonomy_id, t.name, tt.taxonomy, tt.parent, tt.count
FROM wp_term_taxonomy tt
JOIN wp_terms t ON t.term_id = tt.term_id;
```

```
+------------------+---------------+----------+--------+-------+
| term_taxonomy_id | name          | taxonomy  | parent | count |
+------------------+---------------+----------+--------+-------+
|                1 | Uncategorized | category  |      0 |     1 |
|                2 | Technology    | category  |      0 |     3 |
|                3 | WordPress     | category  |      2 |     2 |
|                4 | Tutorial      | post_tag  |      0 |     5 |
|                5 | News          | category  |      0 |     1 |
+------------------+---------------+----------+--------+-------+
```

> **Note:** "WordPress" has `parent = 2` (Technology), making it a child category.

---

### `wp_term_relationships`

Many-to-many join table that links posts (or any object) to term/taxonomy pairs.

| Column | Type | Description |
|---|---|---|
| `object_id` | BIGINT UNSIGNED | FK → `wp_posts.ID` (or any object) |
| `term_taxonomy_id` | BIGINT UNSIGNED | FK → `wp_term_taxonomy.term_taxonomy_id` |
| `term_order` | INT | Sort order within the relationship |

**Sample data:**

```sql
SELECT tr.object_id, t.name, tt.taxonomy
FROM wp_term_relationships tr
JOIN wp_term_taxonomy tt ON tt.term_taxonomy_id = tr.term_taxonomy_id
JOIN wp_terms t ON t.term_id = tt.term_id
WHERE tr.object_id = 1;
```

```
+-----------+---------------+----------+
| object_id | name          | taxonomy |
+-----------+---------------+----------+
|         1 | Uncategorized | category |
|         1 | Tutorial      | post_tag |
+-----------+---------------+----------+
```

---

### `wp_termmeta`

Stores key/value pairs for any term (added in WordPress 4.4).

| Column | Type | Description |
|---|---|---|
| `meta_id` | BIGINT UNSIGNED | Primary key |
| `term_id` | BIGINT UNSIGNED | FK → `wp_terms.term_id` |
| `meta_key` | VARCHAR(255) | Key name |
| `meta_value` | LONGTEXT | Value |

**Sample data:**

```sql
SELECT meta_id, term_id, meta_key, meta_value
FROM wp_termmeta
WHERE term_id = 2;
```

```
+---------+---------+------------------+-------------------+
| meta_id | term_id | meta_key         | meta_value        |
+---------+---------+------------------+-------------------+
|       1 |       2 | category_icon    | dashicons-desktop |
|       2 |       2 | category_color   | #0073aa           |
+---------+---------+------------------+-------------------+
```

---

### `wp_comments`

Stores every comment, trackback, and pingback.

| Column | Type | Description |
|---|---|---|
| `comment_ID` | BIGINT UNSIGNED | Primary key |
| `comment_post_ID` | BIGINT UNSIGNED | FK → `wp_posts.ID` |
| `comment_author` | TINYTEXT | Commenter name |
| `comment_author_email` | VARCHAR(100) | Commenter email |
| `comment_author_url` | VARCHAR(200) | Commenter URL |
| `comment_author_IP` | VARCHAR(100) | Commenter IP address |
| `comment_date` | DATETIME | Date posted (site timezone) |
| `comment_date_gmt` | DATETIME | Date posted (UTC) |
| `comment_content` | TEXT | Comment body |
| `comment_karma` | INT | Legacy voting field |
| `comment_approved` | VARCHAR(20) | `1` (approved), `0` (pending), `spam`, `trash` |
| `comment_agent` | VARCHAR(255) | Browser user agent |
| `comment_type` | VARCHAR(20) | `comment`, `trackback`, `pingback`, or custom |
| `comment_parent` | BIGINT UNSIGNED | FK → `wp_comments.comment_ID` (for threaded replies) |
| `user_id` | BIGINT UNSIGNED | FK → `wp_users.ID` (0 for guests) |

**Sample data:**

```sql
SELECT comment_ID, comment_post_ID, comment_author, comment_approved, comment_type
FROM wp_comments;
```

```
+------------+-----------------+----------------+------------------+--------------+
| comment_ID | comment_post_ID | comment_author | comment_approved | comment_type |
+------------+-----------------+----------------+------------------+--------------+
|          1 |               1 | Jane Doe       | 1                | comment      |
|          2 |               1 | John Smith     | 1                | comment      |
|          3 |               1 | Spammer        | spam             | comment      |
|          4 |               1 | Jane Doe       | 0                | comment      |
+------------+-----------------+----------------+------------------+--------------+
```

---

### `wp_commentmeta`

Stores key/value pairs for any comment.

| Column | Type | Description |
|---|---|---|
| `meta_id` | BIGINT UNSIGNED | Primary key |
| `comment_id` | BIGINT UNSIGNED | FK → `wp_comments.comment_ID` |
| `meta_key` | VARCHAR(255) | Key name |
| `meta_value` | LONGTEXT | Value |

**Sample data:**

```sql
SELECT meta_id, comment_id, meta_key, meta_value
FROM wp_commentmeta
WHERE comment_id = 1;
```

```
+---------+------------+--------------+-------------+
| meta_id | comment_id | meta_key     | meta_value  |
+---------+------------+--------------+-------------+
|       1 |          1 | rating       | 5           |
|       2 |          1 | akismet_result | ham       |
+---------+------------+--------------+-------------+
```

---

### `wp_options`

Site-wide key/value store for settings, plugin data, and transients.

| Column | Type | Description |
|---|---|---|
| `option_id` | BIGINT UNSIGNED | Primary key |
| `option_name` | VARCHAR(191) | Unique key (e.g. `siteurl`, `blogname`) |
| `option_value` | LONGTEXT | Value (may be serialised PHP) |
| `autoload` | VARCHAR(20) | `yes` = loaded on every page; `no` = loaded on demand |

**Important built-in options:**

```sql
SELECT option_name, option_value
FROM wp_options
WHERE option_name IN (
    'siteurl', 'blogname', 'blogdescription',
    'admin_email', 'blogpublic', 'active_plugins'
);
```

```
+---------------------+--------------------------------------------------+
| option_name         | option_value                                     |
+---------------------+--------------------------------------------------+
| siteurl             | https://example.com                              |
| blogname            | My WordPress Site                                |
| blogdescription     | Just another WordPress site                      |
| admin_email         | admin@example.com                                |
| blogpublic          | 1                                                |
| active_plugins      | a:1:{i:0;s:19:"my-plugin/plugin.php";}           |
+---------------------+--------------------------------------------------+
```

> **Performance tip:** Keep the number of `autoload = yes` options small. All autoloaded options are loaded into memory on every page request.

---

### `wp_links`

Legacy blogroll links (added pre-WordPress 3.5, hidden by default today).

| Column | Type | Description |
|---|---|---|
| `link_id` | BIGINT UNSIGNED | Primary key |
| `link_url` | VARCHAR(255) | The URL |
| `link_name` | VARCHAR(255) | Display name |
| `link_image` | VARCHAR(255) | Optional image URL |
| `link_target` | VARCHAR(25) | `_blank`, `_top`, etc. |
| `link_description` | VARCHAR(255) | Short description |
| `link_visible` | VARCHAR(20) | `Y` (visible) or `N` (hidden) |
| `link_owner` | BIGINT UNSIGNED | FK → `wp_users.ID` |
| `link_rating` | INT | 0–10 rating |
| `link_updated` | DATETIME | Last update timestamp |
| `link_rel` | VARCHAR(255) | XFN relationship |
| `link_notes` | MEDIUMTEXT | Long notes |
| `link_rss` | VARCHAR(255) | RSS feed URL |

---

## Table Relationships

```
wp_users ──────────────────────────────┐
  │ ID                                 │ link_owner
  │                          wp_links ─┘
  │ 1:N via user_id
  ├──► wp_usermeta (user_id)
  │
  │ 1:N via post_author
  └──► wp_posts ──────────────────────────────────────────┐
         │ ID                                             │
         │                                               │
         │ 1:N via post_id                               │ 1:N via comment_post_ID
         ├──► wp_postmeta (post_id)                      │
         │                                               ▼
         │ M:N via object_id + term_taxonomy_id    wp_comments ──── 1:N ──► wp_commentmeta
         └──► wp_term_relationships                   │ comment_ID       (comment_id)
                │ term_taxonomy_id                    │
                ▼                                     │ self-join (comment_parent)
          wp_term_taxonomy ── 1:1 ──► wp_terms        └──► wp_comments (replies)
            │ term_id                  │ term_id
            │                         │ 1:N via term_id
            │ self-join (parent)       └──► wp_termmeta (term_id)
            └──► wp_term_taxonomy
```

### Quick Relationship Reference

| Child table | FK column | Parent table | PK column |
|---|---|---|---|
| `wp_posts` | `post_author` | `wp_users` | `ID` |
| `wp_posts` | `post_parent` | `wp_posts` | `ID` |
| `wp_postmeta` | `post_id` | `wp_posts` | `ID` |
| `wp_usermeta` | `user_id` | `wp_users` | `ID` |
| `wp_comments` | `comment_post_ID` | `wp_posts` | `ID` |
| `wp_comments` | `user_id` | `wp_users` | `ID` |
| `wp_comments` | `comment_parent` | `wp_comments` | `comment_ID` |
| `wp_commentmeta` | `comment_id` | `wp_comments` | `comment_ID` |
| `wp_term_relationships` | `object_id` | `wp_posts` | `ID` |
| `wp_term_relationships` | `term_taxonomy_id` | `wp_term_taxonomy` | `term_taxonomy_id` |
| `wp_term_taxonomy` | `term_id` | `wp_terms` | `term_id` |
| `wp_term_taxonomy` | `parent` | `wp_term_taxonomy` | `term_taxonomy_id` |
| `wp_termmeta` | `term_id` | `wp_terms` | `term_id` |
| `wp_links` | `link_owner` | `wp_users` | `ID` |

---

## Common Queries

### Get all published posts with author name

```sql
SELECT
    p.ID,
    p.post_title,
    p.post_date,
    u.display_name AS author
FROM wp_posts p
JOIN wp_users u ON u.ID = p.post_author
WHERE p.post_type   = 'post'
  AND p.post_status = 'publish'
ORDER BY p.post_date DESC;
```

---

### Get posts with a specific category

```sql
SELECT
    p.ID,
    p.post_title,
    t.name AS category
FROM wp_posts p
JOIN wp_term_relationships tr ON tr.object_id = p.ID
JOIN wp_term_taxonomy tt      ON tt.term_taxonomy_id = tr.term_taxonomy_id
JOIN wp_terms t               ON t.term_id = tt.term_id
WHERE tt.taxonomy    = 'category'
  AND t.slug         = 'technology'
  AND p.post_status  = 'publish';
```

---

### Get all tags for a post

```sql
SELECT t.name, t.slug
FROM wp_terms t
JOIN wp_term_taxonomy tt      ON tt.term_id = t.term_id
JOIN wp_term_relationships tr ON tr.term_taxonomy_id = tt.term_taxonomy_id
WHERE tr.object_id = 1          -- post ID
  AND tt.taxonomy  = 'post_tag';
```

---

### Get a custom field value for a post

```sql
SELECT meta_value
FROM wp_postmeta
WHERE post_id  = 1
  AND meta_key = '_price';
```

---

### Get a post with ALL its custom fields

```sql
SELECT
    p.ID,
    p.post_title,
    pm.meta_key,
    pm.meta_value
FROM wp_posts p
LEFT JOIN wp_postmeta pm ON pm.post_id = p.ID
WHERE p.ID = 1
ORDER BY pm.meta_key;
```

---

### Get all approved comments for a post

```sql
SELECT
    c.comment_ID,
    c.comment_author,
    c.comment_date,
    c.comment_content,
    c.comment_parent
FROM wp_comments c
WHERE c.comment_post_ID  = 1
  AND c.comment_approved = '1'
ORDER BY c.comment_date ASC;
```

---

### Get all users with a specific role

```sql
SELECT
    u.ID,
    u.user_login,
    u.user_email,
    u.display_name
FROM wp_users u
JOIN wp_usermeta um ON um.user_id = u.ID
WHERE um.meta_key   = 'wp_capabilities'
  AND um.meta_value LIKE '%editor%';
```

---

### Get posts with their featured image URL

```sql
SELECT
    p.ID,
    p.post_title,
    att.guid AS featured_image_url
FROM wp_posts p
-- Step 1: get the attachment ID stored in _thumbnail_id
JOIN wp_postmeta thumb ON thumb.post_id = p.ID
                       AND thumb.meta_key = '_thumbnail_id'
-- Step 2: get the attachment post
JOIN wp_posts att ON att.ID = thumb.meta_value
                  AND att.post_type = 'attachment'
WHERE p.post_type   = 'post'
  AND p.post_status = 'publish';
```

---

### Count posts per category

```sql
SELECT
    t.name          AS category,
    tt.count        AS post_count
FROM wp_terms t
JOIN wp_term_taxonomy tt ON tt.term_id = t.term_id
WHERE tt.taxonomy = 'category'
ORDER BY tt.count DESC;
```

---

### Find all child pages of a parent page

```sql
SELECT ID, post_title, post_status
FROM wp_posts
WHERE post_parent = 2          -- parent page ID
  AND post_type   = 'page'
ORDER BY menu_order ASC;
```

---

### Get a site option

```sql
-- Single option
SELECT option_value
FROM wp_options
WHERE option_name = 'blogname';

-- Multiple options at once
SELECT option_name, option_value
FROM wp_options
WHERE option_name IN ('siteurl', 'blogname', 'admin_email');
```

---

### Search posts by custom field (meta query)

```sql
-- Products where price < 50
SELECT p.ID, p.post_title, pm.meta_value AS price
FROM wp_posts p
JOIN wp_postmeta pm ON pm.post_id = p.ID
                    AND pm.meta_key = '_price'
WHERE p.post_type   = 'product'
  AND p.post_status = 'publish'
  AND CAST(pm.meta_value AS DECIMAL(10,2)) < 50
ORDER BY CAST(pm.meta_value AS DECIMAL(10,2)) ASC;
```

---

### Get comment count per post

```sql
SELECT
    p.ID,
    p.post_title,
    COUNT(c.comment_ID) AS comment_count
FROM wp_posts p
LEFT JOIN wp_comments c ON c.comment_post_ID = p.ID
                        AND c.comment_approved = '1'
WHERE p.post_type   = 'post'
  AND p.post_status = 'publish'
GROUP BY p.ID
ORDER BY comment_count DESC;
```

---

## Helpful Resources

- [WordPress Database Description (Official)](https://codex.wordpress.org/Database_Description)
- [WordPress wpdb Class Reference](https://developer.wordpress.org/reference/classes/wpdb/)
- [WordPress Schema on GitHub](https://github.com/WordPress/WordPress/blob/master/wp-admin/includes/schema.php)
