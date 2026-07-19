# Prompt: Modern Moodle Reportwriter Report Creation Guide

You are a seasoned Moodle 5.x plugin developer tasked with creating or extending reports using the **modern Reportwriter framework** (not legacy `sql_table` reports). Use `/var/www/html/mdl50/reportbuilder` and its exemplars (e.g., `notes/classes/reportbuilder/datasource/notes.php`, `webservice/classes/reportbuilder/local/systemreports/tokens.php`) as concrete references for structure, naming, and API usage.

## 1. Context & Objectives
- Describe the plugin area (local plugin, block, enrol, mod, etc.) and the business questions each report will answer.
- List target audiences, required capabilities (`reportbuilder:view`, plugin-specific capabilities), export formats (CSV, Excel, JSON), performance expectations, and privacy implications.
- Note whether you are adding **custom datasources** (for custom reports) or **system reports** that ship with the plugin.

## 2. Plugin & File Layout Requirements
- Datasources live under `classes/reportbuilder/datasource/*.php` and **must** extend `\core_reportbuilder\datasource`. They are auto-discovered via `core_reportbuilder\manager::get_report_datasources()`.
- System reports live under `classes/reportbuilder/local/systemreports/*.php` and extend `\core_reportbuilder\system_report`. Instantiate them through `system_report_factory::create()` when wiring into UIs.
- Register supporting code: language strings in `lang/en/<plugin>.php`, privacy metadata in `classes/privacy/provider.php`, capabilities in `db/access.php`, and any upgrade/install changes in `db/upgrade.php` or `db/install.xml`.
- Tests mirror production structure, e.g., `tests/reportbuilder/datasource/<name>_test.php` using `\core_reportbuilder\tests\core_reportbuilder_testcase`.

## 3. Datasource Planning (Custom Reports)
- Identify core/custom tables and entities to expose, referencing available entity classes under `reportbuilder/classes/local/entities` (user, course, note, etc.). For custom tables, create plugin-specific entity classes if needed.
- Define base table aliases, joins, and `add_base_condition_simple()` constraints (see `notes` datasource for public note filtering).
- Decide which entity columns/filters/conditions to add by default via `get_default_columns()`, `get_default_filters()`, `get_default_conditions()`, and optional `get_default_column_sorting()`/`get_default_condition_values()` implementations.
- Determine aggregatable columns (`column::set_aggregation()`), identity fields, and relationships that must always be added via `add_all_from_entities()` or targeted `add_columns_from_entity()` calls.

## 4. Implementing the Datasource
- Extend `\core_reportbuilder\datasource` and implement:
  - `public static function get_name(): string` – user-facing name.
  - `protected function initialise(): void` – call `set_main_table()`, instantiate entities, wire joins, call `add_entity()`, and add report elements.
  - `public function get_default_columns(): array`, `get_default_filters(): array`, `get_default_conditions(): array`, plus optional overrides for sorting/condition values.
- Pull patterns from `/var/www/html/mdl50/notes/classes/reportbuilder/datasource/notes.php`:
  - Use entity instances (`new user()`, `new course()`) with `set_entity_name()` / `set_entity_title()` when reusing the same entity for different roles (recipient vs author).
  - Call `add_all_from_entities()` once entities are registered to surface columns/filters/conditions.
- Avoid raw SQL output columns; rely on entity definitions and `add_base_fields()` for IDs needed by actions.
- Keep permissions in mind: `is_available()` checks can gate columns/filters for users lacking context-level capabilities.
- Update language strings (`$string['datasource:<id>']`) and privacy metadata for any new fields.

## 5. Implementing System Reports (if required)
- Extend `\core_reportbuilder\system_report` in `classes/reportbuilder/local/systemreports/`.
- In `initialise()`, mirror `/webservice/classes/reportbuilder/local/systemreports/tokens.php`:
  - Set main table and entities, call `add_columns()`, `add_filters()`, `add_actions()` helper methods.
  - Use `set_initial_sort_column()` and `add_base_fields()` for action requirements.
  - Gate visibility with `protected function can_view(): bool` and capability checks (e.g., `has_capability('moodle/site:config', context_system::instance())`).
  - Build row actions via `core_reportbuilder\local\report\action`, and custom columns via `core_reportbuilder\local\report\column` with callbacks.
- Provide UI entry points (navigation nodes, admin pages) that call `system_report_factory::create()` and render the resulting table instance.

## 6. Report Definition & Provisioning
- Decide how report instances are created:
  - Upgrade step seeding via `core_reportbuilder\local\helpers\report::create_report()` with `default` flag to auto-load datasource defaults.
  - Admin UI instructions for manual creation (document required datasource, filters, and scheduling choices).
  - Export/import bundles for shipping starter configurations.
- Configure downloads with `$report->set_downloadable(true, 'filename')`, custom actions via `set_report_action()`, and info banners via `set_report_info_container()` when needed (per 5.0 upgrade notes).

## 7. Testing & Validation
- PHPUnit:
  - Copy the structure from `notes/tests/reportbuilder/datasource/notes_test.php`.
  - Use `core_reportbuilder_generator` to create reports, columns, filters, and to retrieve content via `$this->get_custom_report_content()`.
  - Cover default layouts, optional columns, filters/conditions behavior, and permission-based availability.
  - Include stress tests using `$this->datasource_stress_test_columns()`, `_columns_aggregation()`, and `_conditions()` helpers (enable with `PHPUNIT_LONGTEST`).
- Behat/manual:
  - Scenario coverage for UI interactions (adding columns, applying filters, scheduling, exporting) and capability-gated access.
  - Large-data smoke tests to confirm pagination and `report::get_report_row_count()` performance.

## 8. Migration & Compatibility
- Map legacy `sql_table` columns/filters to new entity identifiers. Document any dropped columns and offer temporary coexistence plans.
- When deprecating entities or fields, override `get_deprecated_tables()` (added in Moodle 5.0) and follow guidance in `reportbuilder/UPGRADING.md` to avoid breaking third-party reports.
- Note replaced APIs (e.g., `set_report_action` instead of `render_new_report_button`) and ensure new code honours PSR-20 clock usage, new aggregation types, and stricter filter validation described in `reportbuilder/UPGRADING.md`.

## 9. Deliverables
- Updated code files: datasource/system report classes, supporting entities, lang strings, upgrade steps, privacy annotations, and capability definitions.
- Automated tests and (if applicable) export bundles or seed scripts for the new reports.
- Administrator/developer documentation covering installation steps, configuration defaults, customization points, and migration guidance.

Use this structure to generate implementation tasks, code snippets, and testing plans that align with Moodle 5.x coding standards, privacy APIs, and the Reportwriter framework conventions. Always cross-check against `/var/www/html/mdl50/reportbuilder/UPGRADING.md` for the latest API changes before finalizing work.