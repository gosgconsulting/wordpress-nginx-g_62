# WordPress GraphQL Plugin Error Fix

## Error Analysis
The TypeError "Cannot read properties of undefined (reading 'length')" indicates a JavaScript conflict between GraphQL plugins and your WordPress theme or other plugins.

## Immediate Solutions

### Solution 1: Disable Conflicting Plugins via Database
1. Access your WordPress database via phpMyAdmin or command line
2. Navigate to the `wp_options` table
3. Find the row with `option_name = 'active_plugins'`
4. Temporarily modify the `option_value` to remove GraphQL plugins:
   - Remove: `wp-graphql/wp-graphql.php`
   - Remove: `wp-graphql-acf/wp-graphql-acf.php`
   - Remove: `graphql-ide/graphql-ide.php`

### Solution 2: Plugin Deactivation via wp-cli (Docker)
```bash
# Access WordPress container
docker-compose exec wordpress bash

# Deactivate GraphQL plugins
wp plugin deactivate wp-graphql --allow-root
wp plugin deactivate wp-graphql-acf --allow-root
wp plugin deactivate graphql-ide --allow-root
```

### Solution 3: Manual Plugin Directory Rename
1. Access your WordPress files via Docker volume
2. Navigate to `/var/www/wp-content/plugins/`
3. Rename plugin directories:
   - `wp-graphql` → `wp-graphql-disabled`
   - `wp-graphql-acf` → `wp-graphql-acf-disabled`
   - `graphql-ide` → `graphql-ide-disabled`

## Root Cause Fixes

### Fix 1: Plugin Compatibility Check
The error suggests GraphQL ACF plugin is trying to access an undefined array. Add this to your theme's `functions.php`:

```php
// GraphQL ACF compatibility fix
add_action('init', function() {
    if (class_exists('WPGraphQL\ACF\ACFLoader')) {
        // Ensure ACF fields are properly initialized before GraphQL
        if (function_exists('acf_get_field_groups')) {
            $field_groups = acf_get_field_groups();
            if (empty($field_groups)) {
                // Prevent GraphQL ACF from processing empty field groups
                remove_action('graphql_register_types', ['WPGraphQL\ACF\ACFLoader', 'register_types']);
            }
        }
    }
}, 1);
```

### Fix 2: JavaScript Conflict Resolution
Add this to your theme's `functions.php` to prevent JS conflicts:

```php
// Prevent GraphQL IDE conflicts
add_action('wp_enqueue_scripts', function() {
    if (is_admin()) {
        return;
    }
    
    // Dequeue GraphQL IDE scripts on frontend
    wp_dequeue_script('graphql-ide');
    wp_dequeue_style('graphql-ide');
}, 100);
```

## Recovery Steps
1. Enable debug mode (already done in docker-compose.yml)
2. Restart WordPress container: `docker-compose restart wordpress`
3. Check error logs: `docker-compose logs wordpress`
4. Apply one of the solutions above
5. Test site functionality
6. Re-enable plugins one by one to identify the specific conflict

## Prevention
- Always test GraphQL plugins in staging environment
- Keep ACF and GraphQL plugins updated
- Use plugin conflict detection tools
- Monitor JavaScript console for errors after plugin installations
