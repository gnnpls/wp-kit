# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`gnnpls/wp-kit` — a dependency-free PHP 7.4+ library (namespace `Gnnpls\WP`, hook/error prefix `mwp_`). A consumer declares a WordPress feature once as a `Resource` of `Operation`s with `Field`s, and the three adapters project it onto REST routes, WP-CLI commands and the WordPress Abilities API. Hosted at `https://github.com/gnnpls/wp-kit` and published on Packagist. Public identity is Nikos Giannopoulos <nikos@gnnpls.com>, https://gnnpls.com — no references to the previous vendor in files, docs or examples (use `my-plugin`). The `mwp_` prefix is public API and stays.

There is no test suite, no PHPUnit config and no dev tooling in this repo. `.github/workflows/lint.yml` only runs `php -l` on every tracked PHP file (PHP 7.4 and 8.3) and `composer validate --strict`.

## Commands

Lint everything CI lints:

```bash
git ls-files -z -- "*.php" | xargs -0 -n1 php -l
```

Validate composer.json as CI does:

```bash
composer validate --strict --no-check-lock
```

Everything in `src/` is guarded by `if (!defined('ABSPATH')) exit;` and `bootstrap.php` returns early outside WordPress, so nothing here can be executed standalone. Behavioural verification happens in a consumer plugin (e.g. under a DDEV site) — run its PHPUnit/parity tests against a local checkout of this package.

## Release checklist

Version lives in two places that must be bumped together: `version.php` (the runtime value read by `bootstrap.php`) and `CHANGELOG.md`. `composer.json` carries no `version` field: push a `vX.Y.Z` tag to GitHub and Packagist picks it up. New public methods get `@since <version>`.

## Architecture

### Loading is version-gated, not Composer-driven

`composer.json` autoloads only `bootstrap.php` (`autoload.files`); the PSR-4 map is `autoload-dev` only, on purpose. Several active plugins may each ship their own `vendor/gnnpls/wp-kit`; every copy calls `Kit::register($version, $dir)`, and on `plugins_loaded` priority -100 `Kit::boot()` picks the highest version, installs *that* copy's `spl_autoload` for `Gnnpls\WP\*` and flushes `Kit::on_ready()` callbacks. Consequences:

- `src/Kit.php` is loaded by whichever plugin includes it first, even if a newer copy later wins the autoloader. Its public API must stay backwards compatible forever; never remove or change the signature of a `Kit` method.
- Consumers must do all kit work inside `Kit::on_ready()` — nothing in `Gnnpls\WP` is autoloadable before boot.
- A copy registered after `plugins_loaded` has already fired (or outside WordPress) boots immediately; the first `register()` decides the boot timing, later copies only join the version race if boot hasn't happened yet.

### Declaration → execution → projection

```
Resource (abstract, one per feature)
  ├─ name(), service(), operations()           required
  ├─ rest_namespace()/cli_base()/ability_category()  null = surface disabled for whole resource
  ├─ capability()                              default resolver ('manage_options')
  └─ surfaces()/surfaces_reason()              default surface set (0.2.0)
        │  ops() binds each Operation (attach name+resource, apply_defaults) and memoises
        ▼
Operation (final, fluent) ── read()/write()/destructive() factories
  ├─ input([Field...]), capability(), confirm(), surfaces()
  ├─ rest(method,path), cli(name,opts), ability(name)   per-surface bindings
  └─ run(array $input, Context $ctx)           THE single execution path
        │
        ▼
Adapters/{Rest,Cli,Ability}_Adapter  ── translate transport in/out around run()
Field_Map                            ── one Field → REST args / CLI synopsis / JSON Schema
Inventory::export(Registry)          ── flat rows for parity tests (surfaces + excluded + why)
```

Rules that follow from this shape:

- **Adapters never contain business rules.** Validation, authorisation, confirmation and the `mwp_operation_*` hooks live in `Operation::run()` (normalise → validate → `Capability` → `Confirm` → before-run action → `invoke()` → result filter). If a behaviour would differ per surface, it belongs in `Context` (`surface()`, `is_unattended_cli()`), not in an adapter.
- **A parameter is declared once.** Add field types/modifiers in `Field` and project them in `Field_Map` for all three surfaces in the same change; the README's "What one field becomes" table is the contract.
- **Advertised ⇒ enforced.** Any constraint a field projects into REST args or the ability schema must also be checked in `Field` validation, including on nested `Field::object()->properties([...])` children (0.3.0 fixed constraints that were advertised but never validated). Adding a modifier to `Field_Map` without validation is a security bug.
- **`Field` is an immutable value object** (setters return clones); `Operation` is mutable and bound to exactly one `Resource` via `attach()`.
- **Surface opt-out has two levels.** `Operation::surfaces()` (sets `surfaces_set`) beats `Resource::surfaces()`; both accept a callable resolved lazily (CLI adapter registers on `init` priority 5 for this reason). Any exclusion should carry a reason so `Inventory` can distinguish deliberate omission from a missing registration.
- **Service call mapping**: `invoke()` matches service method parameters by name via reflection — but only for declared fields (and `confirm`); undeclared keys admitted by `->allow_extra()` never bind by name and go to the first `array`-typed parameter; `->args(callable)` overrides.
- **Handler exceptions become `mwp_operation_failed` (500) with a generic message.** The real exception is `error_log`ged and exposed as `detail` only on the CLI surface or with `WP_DEBUG`; never put exception text back into REST/ability responses.
- **Unattended WP-CLI (no `--user`) is trusted as admin** — `Capability::check()` short-circuits on `Context::is_unattended_cli()`. Don't "fix" this.
- **Destructive ops default to requiring `confirm: true` on the Ability surface only**; CLI maps `--yes` onto the same field. `Confirm::check()` coerces the flag like a `bool` field, so `"no"`/`"off"`/arbitrary strings do not confirm.

### Hooks

All custom hooks are `mwp_`-prefixed and documented WP-core style above each `do_action`/`apply_filters`. `HOOKS.md` is the authoritative declaration (name, type, params, where fired) and the README "Hooks" table is a summary of it — update both when adding or changing a hook. Error codes follow the same prefix: `mwp_invalid_param`, `mwp_forbidden` (401 anonymous / 403 authenticated), `mwp_confirm_required`, `mwp_no_handler`, `mwp_operation_failed`.

## Conventions specific to this repo

- PHP 7.4 compatible: no `readonly`, enums, `match`, named args or union types in `src/`. Typed properties are avoided in favour of `@var` docblocks.
- WordPress-style naming: `Snake_Case` classes, `snake_case` methods, 4-space indent, one class per file at `src/<Class>.php` (the loader's autoloader maps `Gnnpls\WP\Adapters\X` → `src/Adapters/X.php`).
- Every file, class and public method has a docblock with `@since`; every file starts with the `ABSPATH` guard.
- WordPress functions are always guarded with `function_exists()`/`class_exists()` (`do_action`, `apply_filters`, `WP_CLI`, `register_rest_route`, `wp_register_ability`) so the classes can be constructed in isolation.
- `CHANGELOG.md` entries record the originating prompt/intent and a "Backwards compatibility" note when relevant.
