# Connections Business Directory — Community Patch

> **Community-maintained security and compatibility patch for Connections Business Directory 10.4.48**

[![License: GPL v2](https://img.shields.io/badge/License-GPL%20v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)
[![Base Version](https://img.shields.io/badge/Base%20Version-10.4.48-lightgrey)](https://github.com/Connections-Business-Directory/Connections)
[![PHP](https://img.shields.io/badge/PHP-7.0%20–%208.2-brightgreen)]()
[![WordPress](https://img.shields.io/badge/WordPress-5.8%20–%207.x-brightgreen)]()

---

## Background

[Connections Business Directory](https://connections-pro.com/) is a WordPress plugin for managing business directories and address books. As of early 2025, the plugin was **removed from the WordPress plugin repository** due to a security vulnerability ([CVE-2024-12885](https://nvd.nist.gov/vuln/detail/CVE-2024-12885)) and has since been **abandoned by its original developer**, Steven A. Zahm.

Many WordPress sites — particularly membership organizations, professional associations, and small businesses — continue to rely on this plugin and cannot migrate away immediately. This repository provides a community-maintained patch that:

- Fixes the CVE-2024-12885 path traversal vulnerability
- Addresses additional security issues identified during a full audit
- Restores compatibility with PHP 8.x and WordPress 7.x

> ⚠️ **This plugin is abandoned upstream. No further security fixes should be expected from the original developer. A long-term migration to an actively maintained alternative is strongly recommended.** This patch is a stopgap for sites that need time to migrate.

---

## What's Fixed

### Security

| Issue | Severity | File |
|---|---|---|
| [CVE-2024-12885](https://nvd.nist.gov/vuln/detail/CVE-2024-12885) — Arbitrary directory deletion via path traversal | Moderate (CVSS 6.5) | `includes/class.filesystem.php` |
| SQL hardening — unparameterized DELETE query using DB-sourced meta IDs | Low | `includes/class.meta.php` |
| Object injection risk — `unserialize()` on legacy DB data in upgrade routines | Low | `includes/inc.upgrade.php` |
| SQL injection in upgrade routines — two raw `UPDATE` queries without `$wpdb->prepare()` | Low | `includes/inc.upgrade.php` |
| XSS — unescaped output in admin settings field renderer | Low | `includes/settings/class.settings-api.php` |

### PHP 8.x / WordPress 7.x Compatibility

| Issue | File |
|---|---|
| Fatal error on entry edit — `set_quality()` method signature incompatible with WP 7.x parent class | `includes/image/editors/class-wp-image-editor-gmagick.php` |
| Plugin blocked from loading — hard-capped at PHP 8.1 and WordPress 6.3 | `connections.php` |

See [CHANGES.md](CHANGES.md) for full technical details of every fix.

---

## Installation

This plugin is not available on the WordPress plugin repository. Install it manually:

1. Download the latest release from the [Releases](../../releases) page
2. In your WordPress admin, go to **Plugins → Add New → Upload Plugin**
3. Upload the downloaded `.zip` file and click **Install Now**
4. Activate the plugin

Alternatively, clone or download this repository and place the folder in your `wp-content/plugins/` directory.

### Updating from the Original Plugin

If you have Connections Business Directory 10.4.48 (or earlier) installed:

1. **Back up your database and files first.** This plugin stores data in custom database tables; always back up before making changes.
2. Deactivate your current installation
3. Delete the existing plugin folder (your data is in the database and will not be lost)
4. Install this version using the steps above
5. Reactivate and verify everything works

---

## Known Remaining Issues

The following were identified during the security audit but are not yet fixed in this patch. Contributions are welcome.

- **`each()` removed in PHP 8.0** — Two calls in `includes/Utility/_collection.php` need replacing with `foreach`
- **`create_function()` removed in PHP 8.0** — Four calls in `includes/Utility/_collection.php` and `includes/Shortcode/class.shortcode.php` need replacing with closures
- **Old-style constructor** in `includes/class.connections-directory.php` — A method named `Connections_Directory()` matching the class name is deprecated in PHP 8+ and should be renamed to `__construct()`
- **Unescaped admin UI output** — The original developer suppressed many escaping warnings in admin-only contexts with `phpcs:ignore`. A comprehensive escaping pass would harden this further.

---

## Migrating Away from This Plugin

If you are using Connections primarily as a **staff directory or member directory**, consider migrating to one of these actively maintained alternatives:

- [Business Directory Plugin](https://wordpress.org/plugins/business-directory-plugin/) — actively maintained, similar feature set
- [Simple Staff List](https://wordpress.org/plugins/simple-staff-list/) — lightweight staff directory
- [WP User Directory](https://wordpress.org/plugins/wp-user-directory/) — member directory tied to WordPress users
- Custom post types with [ACF](https://www.advancedcustomfields.com/) — for full control over data structure

---

## Contributing

Contributions are welcome and encouraged, especially for the known remaining issues listed above.

1. Fork this repository
2. Create a branch: `git checkout -b fix/description-of-fix`
3. Make your changes
4. Test against PHP 8.x and the latest WordPress
5. Submit a pull request with a clear description of what was changed and why

Please include a reference to any CVE, WordPress Trac ticket, or other documentation relevant to your fix.

---

## Versioning

This repository uses the following tagging convention:

- `v10.4.48-original` — the original unmodified plugin files as released by the original developer
- `v10.4.48-p1` — this patch (first community patch of base version 10.4.48)

Future patches of the same base version will increment the patch number (`-p2`, `-p3`, etc.).

---

## License

This plugin is licensed under the [GNU General Public License v2.0](LICENSE.txt), the same license as the original plugin. You are free to use, modify, and distribute this software under the terms of that license.

**Original plugin:** Connections Business Directory by Steven A. Zahm  
**Original source:** https://github.com/Connections-Business-Directory/Connections  
**This repository** contains modifications to the original work. It is not affiliated with or endorsed by the original developer.

---

## Security Disclosures

If you discover a security vulnerability in this patched version, please open a [GitHub Issue](../../issues) marked **[SECURITY]** or contact the maintainers directly before public disclosure. Please do not open public issues for unpatched vulnerabilities.
