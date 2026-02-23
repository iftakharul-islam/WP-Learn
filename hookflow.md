# WordPress Hook Flow - The Complete Guide

> **Think of WordPress as a train.** It makes stops (hooks) along the way. You can board at any stop to do your work. Actions let you **do something** at a stop. Filters let you **change the luggage** (data) passing through.

---

## Actions vs Filters - The Simple Version

| | Action | Filter |
|---|---|---|
| **What it does** | "Hey, something happened - do your thing!" | "Here's some data - change it if you want" |
| **Returns?** | No return needed | MUST return the data |
| **Think of it as** | An event you react to | A data transformer |
| **Register with** | `add_action()` | `add_filter()` |

```php
// ACTION: React to an event (no return)
add_action('save_post', 'send_notification');
function send_notification($post_id) {
    wp_mail('admin@site.com', 'Post saved!', 'Post #' . $post_id);
}

// FILTER: Modify data (MUST return)
add_filter('the_title', 'add_prefix');
function add_prefix($title) {
    return '>> ' . $title;  // Must return!
}
```

**Priority** (default 10): Lower = runs first. Two functions at priority 10 run in the order they were added.

---

## Frontend Page Load - Hook Order

When someone visits your site, WordPress fires hooks in this exact order:

### Phase 1: Loading Plugins & Theme

```
muplugins_loaded          Must-use plugins loaded
    |
plugins_loaded            All plugins loaded (your plugin is ready)
    |
setup_theme               Before theme's functions.php loads
    |
after_setup_theme         After theme is set up
    |
init                      WordPress core is ready (register things here!)
    |  \__ widgets_init   Register widgets/sidebars
    |
wp_loaded                 Everything is loaded (plugins + theme + user)
```

### Phase 2: Running the Query

```
parse_request             URL being parsed
    |
pre_get_posts             Modify the query BEFORE it runs
    |
wp                        Query is done, posts are ready
```

### Phase 3: Rendering the Page

```
template_redirect         Last chance to redirect before showing page
    |
wp_head                   Inside <head> - meta tags, styles
    |  \__ wp_enqueue_scripts   Fires INSIDE wp_head at priority 1
    |  \__ wp_print_styles      Outputs enqueued CSS
    |  \__ wp_print_scripts     Outputs enqueued JS
    |
wp_body_open              Right after <body> opens
    |
loop_start                The Loop begins
    |
the_post                  Each post is set up (fires per post)
    |
loop_end                  The Loop ends
    |
wp_footer                 Before </body> - footer scripts
    |
shutdown                  Last hook - PHP about to end
```

---

## Admin Page Load - Hook Order

Admin shares the same early loading, then goes its own way:

```
plugins_loaded -> init -> wp_loaded
    |
admin_init                First hook on EVERY admin page load
    |
admin_menu                Register your menu pages (fires after admin_init!)
    |
current_screen            Which admin page are we on?
    |
load-{$page}              Your specific admin page is loading
    |
admin_enqueue_scripts     Enqueue admin CSS/JS
    |
admin_head                Inside admin <head>
    |
admin_notices             Show success/error messages
    |
admin_footer              Admin footer
    |
shutdown                  Done
```

> **Common mistake:** Many devs think `admin_menu` fires before `admin_init`. It doesn't. `admin_init` fires first on every admin page load, then `admin_menu` runs after.

---

## AJAX Request Flow

AJAX goes through `admin-ajax.php` with a shorter path:

```
plugins_loaded -> init -> wp_loaded -> admin_init
    |
    |--> wp_ajax_{action}            Logged-in users
    |
    |--> wp_ajax_nopriv_{action}     Non-logged-in users
    |
shutdown
```

```php
// JS sends: action: 'get_prices'

// For logged-in users
add_action('wp_ajax_get_prices', 'handle_get_prices');

// For everyone (including guests)
add_action('wp_ajax_nopriv_get_prices', 'handle_get_prices');

function handle_get_prices() {
    check_ajax_referer('my_nonce', 'security');
    wp_send_json_success(['price' => 99]); // wp_send_json already calls wp_die()
}
```

---

## REST API Flow

```
plugins_loaded -> init -> wp_loaded -> parse_request
    |
rest_api_init             Register your routes HERE
    |
rest_pre_dispatch         Before request is handled
    |
(your callback runs)
    |
rest_post_dispatch        After response, before sending
    |
shutdown
```

```php
add_action('rest_api_init', function () {
    register_rest_route('myplugin/v1', '/items', [
        'methods'             => 'GET',
        'callback'            => 'get_items',
        'permission_callback' => function () {
            return current_user_can('read');
        },
    ]);
});
```

---

## Login & Authentication Flow

```
login_init                Login page starting
    |
login_enqueue_scripts     Login page CSS/JS
    |
login_head                Inside login page <head>
    |
login_form                Inside the login form (add extra fields here)
    |
(user submits credentials)
    |
authenticate              FILTER - verify credentials (add 2FA here!)
    |
    |--> wp_login          Success - user is logged in
    |       \__ login_redirect    FILTER - where to send user after login
    |
    |--> wp_login_failed   Failure - bad credentials
```

---

## Post Save Flow

When a post is created or updated:

```
wp_insert_post_data       FILTER - modify data before DB write
    |
(database INSERT/UPDATE happens here)
    |
transition_post_status    Status change fires FIRST (draft -> publish)
    |  \__ {old}_to_{new}   e.g. draft_to_publish
    |  \__ {new}_{type}     e.g. publish_post
    |
save_post_{post_type}     Fires for your specific post type
    |
save_post                 Fires for ALL post types
    |
wp_insert_post            Post saved to database
    |
wp_after_insert_post      Post + its meta + terms all saved (safest hook)
```

> **Common mistake:** Many devs assume `transition_post_status` fires after `save_post`. It actually fires **before** it. If you need post meta to be saved too, use `wp_after_insert_post`.

```php
// Save custom field when a 'product' is saved
add_action('save_post_product', 'save_product_price', 10, 3);
function save_product_price($post_id, $post, $update) {
    if (defined('DOING_AUTOSAVE') && DOING_AUTOSAVE) return;
    if (!isset($_POST['product_nonce'])) return;
    if (!wp_verify_nonce($_POST['product_nonce'], 'save_product')) return;
    if (!current_user_can('edit_post', $post_id)) return;

    update_post_meta($post_id, '_price', sanitize_text_field($_POST['price']));
}
```

> **Warning:** Never call `wp_update_post()` inside `save_post` - it creates an infinite loop!

---

## Quick Decision Guide

> **"I want to... which hook do I use?"**

### Registering Things

| I want to... | Hook | Type |
|---|---|---|
| Register a custom post type | `init` | Action |
| Register a taxonomy | `init` | Action |
| Register a widget or sidebar | `widgets_init` | Action |
| Register a REST API route | `rest_api_init` | Action |
| Register a shortcode | `init` | Action |
| Register admin settings | `admin_init` | Action |
| Add an admin menu page | `admin_menu` | Action |
| Add a dashboard widget | `wp_dashboard_setup` | Action |
| Schedule a cron job | `init` | Action |

### Loading Scripts & Styles

| I want to... | Hook | Type |
|---|---|---|
| Load CSS/JS on the frontend | `wp_enqueue_scripts` | Action |
| Load CSS/JS in admin | `admin_enqueue_scripts` | Action |
| Load CSS/JS on login page | `login_enqueue_scripts` | Action |
| Add async/defer to a script tag | `script_loader_tag` | Filter |

### Modifying Content

| I want to... | Hook | Type |
|---|---|---|
| Change post content before display | `the_content` | Filter |
| Change post title before display | `the_title` | Filter |
| Change the excerpt | `the_excerpt` | Filter |
| Change the page `<title>` tag | `document_title_parts` | Filter |
| Add something in `<head>` | `wp_head` | Action |
| Add something before `</body>` | `wp_footer` | Action |

### Working with Queries

| I want to... | Hook | Type |
|---|---|---|
| Modify the main query | `pre_get_posts` | Action |
| Change SQL query clauses | `posts_clauses` | Filter |
| Change posts per page | `pre_get_posts` | Action |
| Exclude a category from listings | `pre_get_posts` | Action |

### Handling Posts

| I want to... | Hook | Type |
|---|---|---|
| Do something when a post is saved | `save_post` | Action |
| Do something when MY post type is saved | `save_post_{type}` | Action |
| Modify post data before saving | `wp_insert_post_data` | Filter |
| Do something when a post is trashed | `wp_trash_post` | Action |
| Do something when a post is deleted | `before_delete_post` | Action |
| React to status change (draft->publish) | `transition_post_status` | Action |

### Users & Authentication

| I want to... | Hook | Type |
|---|---|---|
| Run code after user logs in | `wp_login` | Action |
| Custom login validation (2FA, etc.) | `authenticate` | Filter |
| Redirect after login | `login_redirect` | Filter |
| Run code after logout | `wp_logout` | Action |
| Block certain users from logging in | `wp_authenticate_user` | Filter |
| Log failed login attempts | `wp_login_failed` | Action |

### Admin Customization

| I want to... | Hook | Type |
|---|---|---|
| Show admin notice/message | `admin_notices` | Action |
| Add columns to post list table | `manage_{type}_posts_columns` | Filter |
| Add content to custom columns | `manage_{type}_posts_custom_column` | Action |
| Add row actions (Edit/Trash/View) | `post_row_actions` | Filter |
| Add bulk actions | `bulk_actions-{screen}` | Filter |
| Add body class in admin | `admin_body_class` | Filter |

### Emails

| I want to... | Hook | Type |
|---|---|---|
| Change "From" email address | `wp_mail_from` | Filter |
| Change "From" name | `wp_mail_from_name` | Filter |
| Send HTML emails | `wp_mail_content_type` | Filter |

### Media & Uploads

| I want to... | Hook | Type |
|---|---|---|
| Allow/block file types for upload | `upload_mimes` | Filter |
| Change JPEG compression quality | `jpeg_quality` | Filter |
| Modify `<img>` tag attributes | `wp_get_attachment_image_attributes` | Filter |

### Redirects & URLs

| I want to... | Hook | Type |
|---|---|---|
| Redirect before page loads | `template_redirect` | Action |
| Modify any redirect URL | `wp_redirect` | Filter |
| Change the home URL | `home_url` | Filter |
| Change permalink structure | `post_type_link` | Filter |

### Plugin Lifecycle

| I want to... | Use |
|---|---|
| Run code when plugin is activated | `register_activation_hook()` |
| Run code when plugin is deactivated | `register_deactivation_hook()` |
| Clean up when plugin is deleted | `register_uninstall_hook()` |
| Run code that depends on other plugins | `plugins_loaded` |

---

## Common Filters with Examples

### Modify Post Content

```php
add_filter('the_content', 'add_cta_after_content');
function add_cta_after_content($content) {
    if (is_single()) {
        $content .= '<div class="cta">Like this? Share it!</div>';
    }
    return $content;
}
```

### Allow SVG Uploads

```php
add_filter('upload_mimes', 'allow_svg');
function allow_svg($mimes) {
    $mimes['svg'] = 'image/svg+xml';
    return $mimes;
}
```

### Customize Admin Columns

```php
// Add column
add_filter('manage_product_posts_columns', 'add_price_column');
function add_price_column($columns) {
    $columns['price'] = 'Price';
    return $columns;
}

// Fill column
add_action('manage_product_posts_custom_column', 'fill_price_column', 10, 2);
function fill_price_column($column, $post_id) {
    if ($column === 'price') {
        echo get_post_meta($post_id, '_price', true);
    }
}
```

### Change "From" Email

```php
add_filter('wp_mail_from', function () {
    return 'hello@mysite.com';
});

add_filter('wp_mail_from_name', function () {
    return 'My Site';
});
```

### Modify Main Query

```php
add_action('pre_get_posts', 'customize_main_query');
function customize_main_query($query) {
    // Only modify the main query on the frontend
    if (!is_admin() && $query->is_main_query()) {
        // Show 20 posts per page on blog
        if ($query->is_home()) {
            $query->set('posts_per_page', 20);
        }
        // Exclude a category
        if ($query->is_archive()) {
            $query->set('cat', '-5'); // Exclude category ID 5
        }
    }
}
```

### Add Async/Defer to Scripts

```php
// Modern way (WordPress 6.3+) - use the strategy parameter
wp_enqueue_script('my-analytics', '/path/to/file.js', [], '1.0', [
    'strategy' => 'defer',  // or 'async'
    'in_footer' => true,
]);

// Legacy way (older WordPress) - use script_loader_tag filter
add_filter('script_loader_tag', 'add_defer_to_scripts', 10, 3);
function add_defer_to_scripts($tag, $handle, $src) {
    $defer_scripts = ['my-analytics', 'my-tracking'];

    if (in_array($handle, $defer_scripts)) {
        return str_replace(' src=', ' defer src=', $tag);
    }
    return $tag;
}
```

### Custom Login Redirect

```php
add_filter('login_redirect', 'redirect_by_role', 10, 3);
function redirect_by_role($redirect_to, $requested, $user) {
    if (!is_a($user, 'WP_User')) {
        return $redirect_to; // $user can be WP_Error on failed login
    }
    if (in_array('administrator', $user->roles)) {
        return admin_url();
    }
    return home_url('/dashboard/');
}
```

---

## Helpful Resources

- [WordPress Action Reference (Official)](https://developer.wordpress.org/apis/hooks/action-reference/)
- [WordPress Plugin Handbook - Hooks](https://developer.wordpress.org/plugins/hooks/)
- [HookOrder.com - Interactive Hook Visualizer](https://hookorder.com/)
- [Adam Brown - Every WordPress Hook](https://adambrown.info/p/wp_hooks/hook)
- [WP-Kama Hooks Order](https://wp-kama.com/hooks/actions-order)
