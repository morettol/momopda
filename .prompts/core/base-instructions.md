# Core Moodle Development Principles

## Moodle coding style

* Source for these is https://moodledev.io/general/development/policies/codingstyle.
* Moodle coding style is prioritized, then, PSR-12 or PSR-1, in that order.
* Always use "long" php tags. However, to avoid whitespace problems, DO NOT include the closing tag at the very end of the file.
* Maximum Line Length: Aim for 132 characters.
* Wrapping lines: Indent with 4 spaces by default.
* Terminate lines with LF
* Filenames are lowercase only
* Inline comments should begin with an upper case letter and end with  '.', '?' or '!'
* Class and function names are lowercase words, separated by underscores
* In the case of legacy functions (those not placed in classes), names should start with the Frankenstyle prefix and plugin name to avoid conflicts between plugins.
* Variable names are lowercase words, no word separator.
* Constants should always be in upper case, and always start with Frankenstyle prefix and plugin name (in case of activities the module name only for legacy reasons). They should have words separated by underscores.
* Strings: Always use single quotes when a string is literal, or contains a lot of double quotes. Use double quotes when you need to include plain variables or a lot of single quotes.

## Other development principles

**Security**: Validate inputs, use Moodle security functions  
**I18n**: Externalize all strings
**Database**: DML API only, no raw SQL
**Capabilities**: Implement proper permission checking
**Accessibility**: WCAG 2.1 AA compliance

## Backup / Restore

Any plugin that stores data in its own DB table or file area **must** ship backup/restore classes, or that data will silently disappear when a course is backed up and restored.

### Required files

```
backup/moodle2/backup_{component}_subplugin.class.php   # for submission subplugins
backup/moodle2/restore_{component}_subplugin.class.php
# — or for activity modules:
backup/moodle2/backup_activity_{name}_stepslib.php
backup/moodle2/restore_activity_{name}_stepslib.php
```

### What the backup class must do

1. Define `define_submission_subplugin_structure()` (or the activity equivalent).
2. Create a `backup_nested_element` for the DB table rows, sourced with `backup::VAR_PARENTID`.
3. Call `annotate_files()` for every filearea the plugin uses.

### What the restore class must do

1. Define the matching path element pointing at the XML node written by backup.
2. In the processor method, remap IDs using `get_new_parentid('assign')` and `get_mappingid('submission', ...)`.
3. Call `add_related_files()` with the old item ID so files are copied to the new context.

### How to verify

After writing the classes, run a real backup/restore cycle:

```bash
sudo -u www-data php /srv/lms/moodle/admin/cli/backup.php \
    --courseid=<id> --destination=/tmp/backup-test/

sudo -u www-data php /srv/lms/moodle/admin/cli/restore_backup.php \
    --file=/tmp/backup-test/<file>.mbz --categoryid=1
```

Then confirm via DB that rows were inserted with new IDs and files exist in the new context.

### Reference implementation

`/srv/lms/moodle/public/mod/assign/submission/onlinetext/backup/moodle2/` — the closest structural match for submission subplugins.

## Version Information

- Component naming: `{plugintype}_{pluginname}`
- Version format: YYYYMMDDRR.XX, where YYYYMMDD is the date, RR is a release increment, and XX is a micro increment. 
- Maturity levels: ALPHA, BETA, RC, STABLE
