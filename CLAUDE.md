# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Plugin Overview

**Abandoned Cart Lite for WooCommerce** by Tyche Softwares. Tracks abandoned WooCommerce carts and sends automated recovery emails via WP-Cron / Action Scheduler.

- Requires: PHP 7.4+, WordPress 6.3+, WooCommerce 4.0+
- Text domain: `woocommerce-abandoned-cart`

## Build Commands

```bash
# Build JS assets (Gutenberg blocks)
npm run build

# Development watch mode
npm run start

# Lint JavaScript
npm run lint:js

# Lint CSS
npm run lint:css

# Format code
npm run format
```

## PHP Code Standards

```bash
# Run PHPCS against PHP files using the custom ruleset
phpcs --standard=tyche-phpcs.xml path/to/file.php

# Auto-fix where possible
phpcbf --standard=tyche-phpcs.xml path/to/file.php
```

Key rules from `tyche-phpcs.xml`:
- All DB queries must use `$wpdb->prepare()` with proper placeholders
- Never use direct PHP DB classes/functions — use WP abstraction layer (`$wpdb`)
- No short PHP open tags (`<?`), no `goto`, no backtick operators
- Use `wp_remote_get/post()` instead of `curl_*` or `file_get_contents()`
- Check for deprecated WP functions/classes/parameters

## Architecture

### Entry Point
`woocommerce-ac.php` — main plugin file, instantiates the core class `woocommerce_abandon_cart_lite` and registers cron schedules.

### Core Database Tables
- `{prefix}ac_abandoned_cart_history_lite` — abandoned cart records (registered + guest users)
- `{prefix}ac_email_templates_lite` — reminder email templates
- `{prefix}ac_sent_history_lite` — history of reminder emails sent
- `{prefix}ac_guest_abandoned_cart_history_lite` — guest user data captured at checkout

### Key WordPress Options
- `ac_lite_cart_abandoned_time` — cart cut-off time in minutes
- `wcal_enable_cart_emails` — master toggle for sending emails (`'on'`)
- `wcal_add_utm_to_links` — UTM query string appended to recovery links

### Directory Structure

```
includes/
  class-wcal-common.php         # Shared static utility methods (counts, template helpers)
  class-wcal-guest-ac.php       # Guest user cart tracking logic
  class-wcal-webhooks.php       # WooCommerce webhook topics for cart abandoned/recovered events
  class-wcal-delete-handler.php # Handles cart record deletion
  wcal-functions.php            # Global helper functions
  classes/                      # WP_List_Table subclasses for admin UI tabs
    class-wcal-abandoned-orders-table.php
    class-wcal-recover-orders-table.php
    class-wcal-templates-table.php
    class-wcal-product-report-table.php
    class-wcal-dashboard-report.php
    class-wcal-aes.php          # AES-256 encryption for cart recovery links
    class-wcal-aes-counter.php
  admin/
    class-wcal-abandoned-cart-details.php  # Cart detail popup/page
    class-wcal-personal-data-eraser.php    # GDPR data erasure
    class-wcal-personal-data-export.php    # GDPR data export
    class-wcap-pro-settings.php            # Placeholder hooks for Pro version
  frontend/
    class-wcal-frontend.php           # Frontend loader
    class-wcal-checkout-process.php   # Order placement hooks; marks carts recovered
  blocks/
    class-wcal-gdpr-emails-blocks-integration.php  # WooCommerce Blocks GDPR consent
  component/                          # Reusable Tyche Softwares shared components
    plugin-tracking/                  # Usage tracking opt-in/out
    plugin-deactivation/              # Deactivation survey
    upgrade-to-pro/                   # Pro upsell notices/pages
    faq-support/                      # FAQ & Support tab content

cron/
  class-wcal-cron.php           # Email sending logic, runs every 15 min via Action Scheduler

src/                            # JS source (compiled to build/)
  index.js
  wcal-blocks-gdpr-email-compliance/  # Gutenberg block: GDPR consent checkbox
  wcal-blocks-guest-capture.js        # Gutenberg block: guest email capture

views/                          # PHP template files for admin pages
assets/                         # Static CSS, JS, images
```

### Email Recovery Flow
1. Guest/logged-in user adds to cart and abandons (doesn't purchase within cut-off time).
2. `cron/class-wcal-cron.php::wcal_send_email_notification()` runs every 15 min, queries active email templates, and sends reminders via `wp_mail()`.
3. Recovery links use AES-256 encrypted tokens (`includes/classes/class-wcal-aes.php`).
4. When a user clicks a recovery link, `includes/frontend/class-wcal-checkout-process.php` hooks into WooCommerce order completion to mark the cart as recovered.

### Webhook Events
Two custom WooCommerce webhook topics: `wcap_cart.cutoff` (cart abandoned after cut-off) and `wcap_cart.recovered` (order recovered).

## Git Workflow

New features must be branched off `staging`, unless explicitly instructed otherwise.

## PR Requirements (Dangerfile)

All PRs must:
- Have a description body (at least 2 characters)
- Be assigned to an assignee and reviewer
- Be assigned to a milestone
- Reference an issue by including `fix #` in the PR body (case-insensitive, e.g. `Fix #123`)
- Aim to follow `commit_lint` conventions; `todoist` will warn if TODO items are left in the diff
- Not contain `do-not-scan` (skipping PHPCS scan is disallowed)
