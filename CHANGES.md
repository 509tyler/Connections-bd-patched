# Connections Business Directory — Community Patch

**Base version:** 10.4.48  
**PHP compatibility:** 7.0 – 8.2  
**WordPress compatibility:** 5.8 – 7.x  

This patch addresses the security vulnerability that caused the plugin to be removed from the WordPress plugin repository, fixes PHP 8.x compatibility errors, and hardens several additional areas identified during a full security audit. It is intended as a community stopgap for sites that depend on this plugin and cannot migrate away immediately.

> **Note:** Connections Business Directory is abandoned by its original developer and has been removed from the WordPress plugin repository. Use this patched version at your own risk. No further upstream security fixes should be expected. A long-term migration away from this plugin is strongly recommended.

---

## Security Fixes

### CVE-2024-12885 — Arbitrary Directory Deletion (Path Traversal)
**File:** `includes/class.filesystem.php`  
**Severity:** Moderate (CVSS 6.5)

The `cnFileSystem::xrmdir()` method accepted any filesystem path and recursively deleted it without verifying the path was confined to the plugin's image directory. An administrator-level user who could set a crafted entry slug (e.g. containing `../../`) could trigger deletion of arbitrary directories on the server when that entry was deleted.

**Fix:** Added two guards at the top of `xrmdir()`:
1. `realpath()` resolution — collapses any `../` traversal sequences to their true absolute path and returns `false` for non-existent paths, replacing the former `file_exists()` check.
2. Confinement check — verifies the resolved path starts with `realpath(CN_IMAGE_PATH)` before proceeding. If not, the method returns immediately without touching the filesystem.

---

### SQL Injection Hardening
**File:** `includes/class.meta.php`

The `delete()` method built a `DELETE ... WHERE meta_id IN(...)` query by imploding `$meta_ids` directly into the SQL string. Although these values were sourced from a prior `$wpdb->get_col()` call rather than direct user input, the pattern is unsafe by principle and could be exploited if the meta table were compromised by another vector.

**Fix:** All values are now passed through `array_map( 'intval', $meta_ids )` before being imploded into the query string.

---

### Object Injection / SQL Injection in Upgrade Routines
**File:** `includes/inc.upgrade.php`

Seven upgrade migration routines called `unserialize()` directly on data retrieved from legacy database columns (`options`, `addresses`, `phone_numbers`, `email`, `im`, `social`, `websites`). Unsafe deserialization of attacker-controlled data can lead to PHP object injection.

Additionally, two `UPDATE` queries were built using string concatenation without `$wpdb->prepare()`:
- The entry type UPDATE used a value extracted from unserialized data with no sanitization.
- The slug UPDATE used a `sanitize_title()`-processed value but still bypassed `prepare()`.

**Fixes:**
- All seven `unserialize()` calls replaced with `maybe_unserialize()`, which only deserializes if the value is actually a serialized string.
- Added `is_array()` guard before accessing array keys on the deserialized options value.
- Added `sanitize_key()` on the entry type value extracted from deserialized data.
- Both raw `UPDATE` queries converted to `$wpdb->prepare()` with `%s`/`%d` placeholders and explicit `(int)` casts on ID values.

---

### XSS in Admin Settings Output
**File:** `includes/settings/class.settings-api.php`

Two `sprintf()` calls in the settings field renderer output values into HTML without escaping:

1. The `multiselect` field type used `$name`, `$key`, and `$label` directly in attribute and content positions.
2. The `sortable_checklist` field type used `$key` directly in a `<li value="...">` attribute.

While these values are developer-defined plugin option keys rather than direct user input, unescaped output in HTML attribute context is exploitable if option definitions are ever influenced by an attacker (e.g. through a compromised plugin or database).

**Fixes:**
- `multiselect`: wrapped `$name` and `$key` with `esc_attr()`, and `$label` with `esc_html()`.
- `sortable_checklist`: wrapped `$key` with `esc_attr()`.

---

## PHP 8.x Compatibility Fixes

### Fatal Error on Entry Edit — `set_quality()` Signature Mismatch
**File:** `includes/image/editors/class-wp-image-editor-gmagick.php`

WordPress 7.x added a `$dims = []` parameter to `WP_Image_Editor::set_quality()`. PHP 8.0+ enforces that child class method signatures must be compatible with their parent. The `WP_Image_Editor_Gmagick` subclass declared `set_quality( $quality = null )` without the new parameter, causing a fatal error whenever an entry with an image was edited.

**Fix:** Added `$dims = []` to the method signature:
```php
public function set_quality( $quality = null, $dims = [] ) {
```

### Version Gate Bypass
**File:** `connections.php`

The plugin's requirements checker hard-capped supported PHP at `8.1` and WordPress at `6.3`, preventing it from loading on PHP 8.2+ and WordPress 7.x even when the underlying code functions correctly.

**Fix:** Raised the `max` version constraints to permit loading on current PHP and WordPress versions. The plugin now shows an admin notice that it has not been tested beyond its original maximum rather than refusing to activate.

---

## Files Changed

| File | Change |
|---|---|
| `connections.php` | Raised PHP/WP max version constraints |
| `includes/image/editors/class-wp-image-editor-gmagick.php` | Fixed `set_quality()` signature for PHP 8.x / WP 7.x |
| `includes/class.filesystem.php` | **CVE-2024-12885** — path traversal fix in `xrmdir()` |
| `includes/class.meta.php` | SQL hardening — `intval()` cast on meta IDs in DELETE query |
| `includes/inc.upgrade.php` | 8 fixes: `maybe_unserialize()`, `sanitize_key()`, `is_array()` guard, 2× `$wpdb->prepare()` |
| `includes/settings/class.settings-api.php` | XSS hardening — `esc_attr()`/`esc_html()` on unescaped settings output |

---

## Additional Security Notes

A full audit of the plugin identified the following areas that were reviewed but determined to be safe or out of scope for this patch:

- **SQL queries in `class.term.php`** — The broad pattern scan flagged 168+ hits, but manual review confirmed all queries use hardcoded SQL fragments, allowlisted `ORDER BY` values, or `$wpdb->prepare()` correctly. No unsafe queries found.
- **SQL queries in `class.entry-actions.php`** — Status and visibility bulk UPDATE queries use `$wpdb->prepare()` with `%s`/`%d` placeholders. Confirmed safe.
- **AJAX handlers** — All 12 AJAX endpoints were reviewed. CSV export/import handlers use `_validate::ajaxReferer()`. System information handlers check both `current_user_can( 'manage_options' )` and a nonce via `Request\Nonce`. Confirmed safe.
- **XSS in Form field classes** — The `phpcs:ignore EscapeOutput` suppressions throughout `includes/Form/` are legitimate; escaping is applied inside each field's `getHTML()` method using the `_escape` utility class, which wraps `esc_attr()`, `esc_html()`, `wp_kses()`, `safecss_filter_attr()`, and `htmlentities()`.
- **Unserialize in upgrade routines** — The `maybe_unserialize()` fix above addresses this. The affected columns no longer exist in current schema; these routines only run once on very old installs upgrading from pre-2013 data.

---

## Known Remaining Issues

The following were identified during the audit but are out of scope for this patch:

- **104 unescaped `echo` calls** in admin-only UI code — These are suppressed with `phpcs:ignore EscapeOutput` by the original developer and are largely in admin-only contexts where the risk is lower. A comprehensive escaping pass across all admin UI output would be a significant undertaking.
- **Old-style constructor** in `includes/class.connections-directory.php` — A method named `Connections_Directory()` matching the class name is deprecated in PHP 8+ (though not a fatal error in this context). Should be renamed to `__construct()`.
- **`each()` removed in PHP 8.0** — Two calls in `includes/Utility/_collection.php` should be replaced with `foreach`.
- **`create_function()` removed in PHP 8.0** — Four calls in `includes/Utility/_collection.php` and `includes/Shortcode/class.shortcode.php` should be replaced with closures.

Contributions addressing these issues are welcome.
