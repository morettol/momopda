# Course Format Development Patterns and Anti-Patterns

## ⚠️ CRITICAL MOODLE 5.x COURSE FORMAT PATTERNS & BUG PREVENTION

---

### 🚫 Anti-Pattern: Treating format.php as a standalone script

**NEVER DO THIS:**
```php
// ❌ WRONG — format.php is not a standalone page
<?php
require_once('../../config.php');  // Do NOT add this
require_login($course, true);      // Already done by course/view.php
echo $OUTPUT->header();            // Already called by course/view.php
```

**✅ CORRECT — format.php is a code snippet included by course/view.php:**
```php
// ✅ CORRECT — rely on globals already set up by course/view.php
defined('MOODLE_INTERNAL') || die();

$format  = course_get_format($course);  // $course already exists
$course  = $format->get_course();
$context = context_course::instance($course->id);

// Render using the reactive output component
$renderer    = $PAGE->get_renderer('format_[formatname]');
$outputclass = $format->get_output_classname('content');
$widget      = new $outputclass($format);
echo $renderer->render($widget);
```

---

### 🚫 Anti-Pattern: Wrong base class for the format

**NEVER DO THIS:**
```php
// ❌ WRONG — old-style inheritance or missing require
class format_[formatname] extends format_base { ... }
// OR
class format_[formatname] {  // No inheritance at all
```

**✅ CORRECT — always extend core_courseformat\base:**
```php
// ✅ CORRECT
defined('MOODLE_INTERNAL') || die();
require_once($CFG->dirroot . '/course/format/lib.php');

class format_[formatname] extends core_courseformat\base { ... }
```

---

### 🚫 Anti-Pattern: Missing supports_components() → reactive editor broken

**NEVER DO THIS:**
```php
// ❌ WRONG — omitting supports_components() makes the editor fall back to legacy mode
class format_[formatname] extends core_courseformat\base {
    public function supports_ajax() {
        $s = new stdClass();
        $s->capable = true;
        return $s;
    }
    // supports_components() not declared → Moodle 4+ reactive editor disabled
}
```

**✅ CORRECT:**
```php
// ✅ CORRECT — declare both for full Moodle 5.x reactive editor support
public function supports_ajax(): stdClass {
    $s = new stdClass();
    $s->capable = true;
    return $s;
}

public function supports_components(): bool {
    return true;
}
```

---

### 🚫 Anti-Pattern: AMD mutations declared as regular methods

The course editor uses a plugin-merging system that requires mutations to be **class attributes** (arrow function fields), not regular methods. Regular methods are NOT merged properly.

**NEVER DO THIS:**
```javascript
// ❌ WRONG — regular method; won't be picked up by addMutations()
class MyMutations extends DefaultMutations {
    async sectionHighlight(stateManager, sectionIds) { ... }
}
```

**✅ CORRECT — use class field arrow functions:**
```javascript
// ✅ CORRECT
class MyMutations extends DefaultMutations {
    sectionHighlight = async function(stateManager, sectionIds) {
        const course = stateManager.get('course');
        this.sectionLock(stateManager, sectionIds, true);
        const updates = await this._callEditWebservice('section_highlight', course.id, sectionIds);
        stateManager.processUpdates(updates);
        this.sectionLock(stateManager, sectionIds, false);
    };
}
```

---

### 🚫 Anti-Pattern: Returning wrong type from section_action()

`section_action()` must return `null` or an array (never `false`, never a non-array).

**NEVER DO THIS:**
```php
// ❌ WRONG — parent return value discarded; section availability not refreshed
public function section_action($section, $action, $sr) {
    if ($action === 'myaction') {
        // do something
        return; // ← null is OK for custom-only actions, but...
    }
    parent::section_action($section, $action, $sr); // ← return value ignored!
}
```

**✅ CORRECT:**
```php
// ✅ CORRECT
public function section_action($section, $action, $sr): ?array {
    global $PAGE;

    if ($action === 'myaction') {
        // handle custom action
        return null;
    }

    // Always capture and return parent result for standard actions
    $rv = parent::section_action($section, $action, $sr);

    // Refresh the availability element in the response
    $renderer = $PAGE->get_renderer('format_[formatname]');
    if (!($section instanceof section_info)) {
        $section = course_modinfo::instance($this->courseid)->get_section_info($section->section);
    }
    $elementclass   = $this->get_output_classname('content\\section\\availability');
    $availability   = new $elementclass($this, $section);
    $rv['section_availability'] = $renderer->render($availability);

    return $rv;
}
```

---

### 🚫 Anti-Pattern: Storing personal data without a real privacy provider

Course formats almost never need to store personal data. If they do, they need a full GDPR provider. Omitting the provider entirely causes upgrade warnings.

**NEVER DO THIS:**
```php
// ❌ WRONG — no privacy provider at all
// (missing file: classes/privacy/provider.php)
```

**✅ CORRECT — null_provider when no personal data is stored:**
```php
// ✅ CORRECT
namespace format_[formatname]\privacy;
use core_privacy\local\metadata\null_provider;

class provider implements null_provider {
    public static function get_reason(): string {
        return 'privacy:metadata';
    }
}
```

---

### 🚫 Anti-Pattern: Missing inplace_editable free function

Without this function, clicking a section name to edit it in-place will throw a coding exception.

**NEVER DO THIS:**
```php
// ❌ WRONG — lib.php ends without the callback
class format_[formatname] extends core_courseformat\base { ... }
// EOF — inplace_editable callback missing
```

**✅ CORRECT:**
```php
// ✅ CORRECT — add after the class definition in lib.php
function format_[formatname]_inplace_editable($itemtype, $itemid, $newvalue) {
    global $DB, $CFG;
    require_once($CFG->dirroot . '/course/lib.php');
    if ($itemtype === 'sectionname' || $itemtype === 'sectionnamenl') {
        $section = $DB->get_record_sql(
            'SELECT s.* FROM {course_sections} s JOIN {course} c ON s.course = c.id
              WHERE s.id = ? AND c.format = ?',
            [$itemid, '[formatname]'], MUST_EXIST);
        return course_get_format($section->course)
            ->inplace_editable_update_section_name($section, $itemtype, $newvalue);
    }
}
```

---

### 🚫 Anti-Pattern: Wrong restore class name or file location

The restore class filename and class name must match exactly, or backup restore silently skips the plugin.

**NEVER DO THIS:**
```
// ❌ WRONG file name or class name
backup/moodle2/format_[formatname]_restore_plugin.class.php
class format_[formatname]_restore_plugin extends restore_format_plugin { ... }
```

**✅ CORRECT:**
```
// ✅ File: backup/moodle2/restore_format_[formatname]_plugin.class.php
class restore_format_[formatname]_plugin extends restore_format_plugin { ... }
```

---

### ✅ Pattern: Overriding output classes via get_output_classname()

In Moodle 5.x, **never** hard-code output class names — always use the format's auto-resolution:

```php
// ✅ CORRECT — resolves to format_[formatname]\output\courseformat\content if it exists,
// otherwise falls back to core_courseformat\output\local\content
$outputclass = $format->get_output_classname('content');
$widget = new $outputclass($format);
echo $renderer->render($widget);

// For nested classes:
$sectionclass = $format->get_output_classname('content\\section');
$menuclass    = $format->get_output_classname('content\\section\\controlmenu');
```

---

### ✅ Pattern: Copying options when switching formats

When a course switches to your format from another, copy matching options so settings are not lost:

```php
public function update_course_format_options($data, $oldcourse = null): bool {
    $data = (array) $data;
    if ($oldcourse !== null) {
        $oldcourse = (array) $oldcourse;
        foreach (array_keys($this->course_format_options()) as $key) {
            if (!array_key_exists($key, $data) && array_key_exists($key, $oldcourse)) {
                $data[$key] = $oldcourse[$key];
            }
        }
    }
    return $this->update_format_options($data);
}
```

---

### ✅ Pattern: Generating correct section URLs

Use the format's `get_view_url()` rather than building moodle_url manually:

```php
// ✅ CORRECT — respects single-page vs multi-page display, section return logic, etc.
$url = $format->get_view_url($section);

// ✅ For navigation links (opens dedicated section page when applicable):
$url = $format->get_view_url($section, ['navigation' => true]);

// ✅ For AJAX-triggered section return:
$url = $format->get_view_url($section, ['sr' => $sectionreturn]);
```

---

### ✅ Pattern: Section marker (highlight) via formatactions

Moodle 5.x provides `\core_courseformat\formatactions` to manipulate sections:

```php
// ✅ Set a section marker (highlight one section as "current")
$sectioninfo = get_fast_modinfo($course->id)->get_section_info($sectionnumber);
\core_courseformat\formatactions::section($course->id)->set_marker($sectioninfo, true);

// ✅ Remove all markers
\core_courseformat\formatactions::section($course->id)->remove_all_markers();
```

---

### ✅ Pattern: AMD build requirement

After editing any `amd/src/*.js` file, compile with grunt before testing or committing:

```bash
# From the Moodle root:
cd /srv/lms/moodle
npx grunt amd --root=public/course/format/[formatname]

# Or build all AMD in the plugin (requires node/npm):
grunt --gruntfile Gruntfile.js amd
```

The compiled `amd/build/` files **must be committed** — Moodle serves them in production.
