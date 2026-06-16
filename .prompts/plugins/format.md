# Moodle Course Format Plugin Development Guide

You are an expert Moodle developer specializing in course format plugins for Moodle 5.x. Course format plugins control how a course's sections and activities are displayed and navigated.

## Course Format Plugin Architecture

Course formats define the layout and interaction model for course content. In Moodle 5.x, all formats integrate with the **reactive course editor** — a JavaScript state-driven component system built on `core_courseformat`.

### Core Concepts

- **No standalone view.php** — the format's `format.php` is a code snippet *included* by `course/view.php`, not a standalone script
- **Sections** are the primary structural unit; your format controls how they appear and behave
- **Reactive components** — the course editor uses a JavaScript state manager; formats override output classes and AMD modules to customise it
- **No personal data** — formats almost never store personal data; use `null_provider` for GDPR

### File Structure

```
course/format/[formatname]/
├── version.php                                  # Plugin metadata (REQUIRED)
├── lib.php                                      # Main format class (REQUIRED)
├── format.php                                   # Course view snippet (REQUIRED)
├── settings.php                                 # Admin site settings (if needed)
├── lang/
│   └── en/
│       └── format_[formatname].php              # Language strings (REQUIRED)
├── classes/
│   ├── output/
│   │   ├── renderer.php                         # Extends section_renderer
│   │   └── courseformat/
│   │       ├── content.php                      # Content output override
│   │       └── content/
│   │           ├── section.php                  # Section output override
│   │           └── section/
│   │               └── controlmenu.php          # Section control menu override
│   ├── courseformat/
│   │   └── stateactions.php                     # AJAX state action handlers
│   └── privacy/
│       └── provider.php                         # Null privacy provider
├── backup/
│   └── moodle2/
│       └── restore_format_[formatname]_plugin.class.php
├── amd/
│   ├── src/
│   │   ├── mutations.js                         # JS mutations for reactive editor
│   │   └── section.js                           # Section reactive component
│   └── build/                                   # Compiled AMD files (grunt output)
├── db/
│   └── upgrade.php                              # DB upgrades (if format stores data)
└── tests/
    ├── format_[formatname]_test.php
    └── behat/
```

## Core Library Implementation (lib.php)

The main format class **must** extend `core_courseformat\base`:

```php
<?php
defined('MOODLE_INTERNAL') || die();
require_once($CFG->dirroot . '/course/format/lib.php');

class format_[formatname] extends core_courseformat\base {

    /**
     * Return true if this format uses sections.
     */
    public function uses_sections(): bool {
        return true;
    }

    /**
     * Return true to show the course index sidebar.
     */
    public function uses_course_index(): bool {
        return true;
    }

    /**
     * Return true to allow indentation of activities on the course page.
     */
    public function uses_indentation(): bool {
        return (bool) get_config('format_[formatname]', 'indentation');
    }

    /**
     * Display name for a section (user-set name or default).
     */
    public function get_section_name($section): string {
        $section = $this->get_section($section);
        if ((string) $section->name !== '') {
            return format_string($section->name, true,
                ['context' => context_course::instance($this->courseid)]);
        }
        return $this->get_default_section_name($section);
    }

    /**
     * Default section name when none is set by the user.
     */
    public function get_default_section_name($section): string {
        $section = $this->get_section($section);
        if ($section->sectionnum == 0) {
            return get_string('section0name', 'format_[formatname]');
        }
        return get_string('newsection', 'format_[formatname]');
    }

    /**
     * Page title for the course view.
     */
    public function page_title(): string {
        return get_string('sectionoutline');
    }

    /**
     * URL for the course or a specific section.
     *
     * @param int|stdClass|null $section
     * @param array $options  'navigation' (bool), 'sr' (int section return)
     */
    public function get_view_url($section, $options = []): moodle_url {
        $course = $this->get_course();
        $section = (is_null($section) || $section instanceof section_info)
            ? $section
            : $this->get_section($section, IGNORE_MISSING);

        if (array_key_exists('sr', $options)) {
            $pagesection = !is_null($options['sr']) ? $this->get_section($options['sr'], IGNORE_MISSING) : null;
        } else if ($options['navigation'] ?? false) {
            $pagesection = ($section && $section->get_component_instance())
                ? $section->get_component_instance()->get_parent_section()
                : $section;
        } else {
            $pagesection = null;
        }

        if (is_null($pagesection)) {
            $url = new moodle_url('/course/view.php', ['id' => $course->id]);
        } else {
            $url = new moodle_url('/course/section.php', ['id' => $pagesection->id]);
        }

        if ($this->uses_sections() && $section && ($section->id != $pagesection?->id)) {
            $url->set_anchor('section-' . $section->section);
        }

        return $url;
    }

    /**
     * Signal AJAX (reactive editor) support.
     */
    public function supports_ajax(): stdClass {
        $ajaxsupport = new stdClass();
        $ajaxsupport->capable = true;
        return $ajaxsupport;
    }

    /**
     * Signal reactive component support (Moodle 4.0+).
     */
    public function supports_components(): bool {
        return true;
    }

    /**
     * Extend course navigation (optional — remove empty section 0 etc.).
     */
    public function extend_course_navigation($navigation, navigation_node $node): void {
        global $PAGE;
        if ($navigation->includesectionnum === false) {
            $selectedsection = optional_param('section', null, PARAM_INT);
            if ($selectedsection !== null && (!defined('AJAX_SCRIPT') || AJAX_SCRIPT == '0') &&
                    $PAGE->url->compare(new moodle_url('/course/view.php'), URL_MATCH_BASE)) {
                $navigation->includesectionnum = $selectedsection;
            }
        }
        parent::extend_course_navigation($navigation, $node);
    }

    /**
     * Default blocks for a newly created course using this format.
     */
    public function get_default_blocks(): array {
        return [
            BLOCK_POS_LEFT  => [],
            BLOCK_POS_RIGHT => [],
        ];
    }

    /**
     * Format-specific options stored on the course record.
     *
     * @param bool $foreditform  true when building the edit form (add labels/elements)
     */
    public function course_format_options($foreditform = false): array {
        static $courseformatoptions = false;
        if ($courseformatoptions === false) {
            $courseformatoptions = [
                'hiddensections' => [
                    'default' => get_config('moodlecourse', 'hiddensections'),
                    'type'    => PARAM_INT,
                ],
                'coursedisplay' => [
                    'default' => get_config('moodlecourse', 'coursedisplay'),
                    'type'    => PARAM_INT,
                ],
            ];
        }
        if ($foreditform && !isset($courseformatoptions['coursedisplay']['label'])) {
            $courseformatoptions = array_merge_recursive($courseformatoptions, [
                'hiddensections' => [
                    'label'        => new lang_string('hiddensections'),
                    'element_type' => 'select',
                    'element_attributes' => [[
                        0 => new lang_string('hiddensectionscollapsed'),
                        1 => new lang_string('hiddensectionsinvisible'),
                    ]],
                ],
                'coursedisplay' => [
                    'label'        => new lang_string('coursedisplay'),
                    'element_type' => 'select',
                    'element_attributes' => [[
                        COURSE_DISPLAY_SINGLEPAGE => new lang_string('coursedisplay_single'),
                        COURSE_DISPLAY_MULTIPAGE  => new lang_string('coursedisplay_multi'),
                    ]],
                    'help'           => 'coursedisplay',
                    'help_component' => 'moodle',
                ],
            ]);
        }
        return $courseformatoptions;
    }

    /**
     * Persist format options when the course is saved.
     *
     * Copy options from the previous format when switching to this one.
     */
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

    /**
     * Allow teachers to delete sections.
     */
    public function can_delete_section($section): bool {
        return true;
    }

    /**
     * Whether this format supports a news/announcements forum.
     */
    public function supports_news(): bool {
        return true;
    }

    /**
     * Allow "stealth" module visibility (hidden on course page but accessible by URL).
     */
    public function allow_stealth_module_visibility($cm, $section): bool {
        return !$section->section || $section->visible;
    }

    /**
     * Handle AJAX section actions (show/hide, custom actions).
     *
     * @param section_info|stdClass $section
     * @param string $action
     * @param int    $sr  section return
     */
    public function section_action($section, $action, $sr): ?array {
        global $PAGE;

        $rv = parent::section_action($section, $action, $sr);

        $renderer = $PAGE->get_renderer('format_[formatname]');
        if (!($section instanceof section_info)) {
            $section = course_modinfo::instance($this->courseid)->get_section_info($section->section);
        }
        $elementclass = $this->get_output_classname('content\\section\\availability');
        $availability = new $elementclass($this, $section);
        $rv['section_availability'] = $renderer->render($availability);

        return $rv;
    }

    /**
     * Expose format config to external/web-service functions.
     */
    public function get_config_for_external(): array {
        return $this->get_format_options();
    }
}

/**
 * Support for in-place section name editing.
 */
function format_[formatname]_inplace_editable($itemtype, $itemid, $newvalue) {
    global $DB, $CFG;
    require_once($CFG->dirroot . '/course/lib.php');
    if ($itemtype === 'sectionname' || $itemtype === 'sectionnamenl') {
        $section = $DB->get_record_sql(
            'SELECT s.* FROM {course_sections} s JOIN {course} c ON s.course = c.id
              WHERE s.id = ? AND c.format = ?',
            [$itemid, '[formatname]'], MUST_EXIST);
        return course_get_format($section->course)->inplace_editable_update_section_name($section, $itemtype, $newvalue);
    }
}
```

## Course View Snippet (format.php)

This file is **included** (not executed directly) by `course/view.php`. It has access to `$course`, `$marker`, `$displaysection`, `$PAGE`, `$OUTPUT`, and all other globals already set up by the parent script.

```php
<?php
defined('MOODLE_INTERNAL') || die();

require_once($CFG->libdir . '/filelib.php');
require_once($CFG->libdir . '/completionlib.php');

// Retrieve format options and attach to $course.
$format = course_get_format($course);
$course = $format->get_course();
$context = context_course::instance($course->id);

// Handle section marker (highlight) if submitted.
if (($marker >= 0) && has_capability('moodle/course:setcurrentsection', $context) && confirm_sesskey()) {
    $course->marker = $marker;
    if ($marker == 0) {
        \core_courseformat\formatactions::section($course->id)->remove_all_markers();
    } else {
        $sectioninfo = get_fast_modinfo($course->id)->get_section_info($marker);
        \core_courseformat\formatactions::section($course->id)->set_marker($sectioninfo, true);
    }
}

// Ensure section 0 (general section) exists.
course_create_sections_if_missing($course, 0);

$renderer = $PAGE->get_renderer('format_[formatname]');

if (!is_null($displaysection)) {
    $format->set_sectionnum($displaysection);
}

// Render the course content via reactive output components.
$outputclass = $format->get_output_classname('content');
$widget = new $outputclass($format);
echo $renderer->render($widget);
```

## Admin Settings (settings.php)

```php
<?php
defined('MOODLE_INTERNAL') || die;

if ($ADMIN->fulltree) {
    $settings->add(new admin_setting_configcheckbox(
        'format_[formatname]/indentation',
        new lang_string('indentation', 'format_[formatname]'),
        new lang_string('indentation_help', 'format_[formatname]'),
        1
    ));
}
```

## Language File (lang/en/format_[formatname].php)

```php
<?php
$string['currentsection']    = 'Current section';
$string['hidefromothers']    = 'Hide';
$string['indentation']       = 'Allow indentation on course page';
$string['indentation_help']  = 'Allow teachers to indent items on the course page.';
$string['newsection']        = 'New section';
$string['page-course-view-[formatname]']   = 'Any course main page in [formatname] format';
$string['page-course-view-[formatname]-x'] = 'Any course page in [formatname] format';
$string['plugin_description'] = 'The course is divided into [formatname] sections.';
$string['pluginname']        = '[Formatname]';
$string['privacy:metadata']  = 'The [formatname] format plugin does not store any personal data.';
$string['section0name']      = 'General';
$string['sectionname']       = 'Section';
$string['showfromothers']    = 'Show';
```

## Output Classes

### Renderer (classes/output/renderer.php)

```php
<?php
namespace format_[formatname]\output;

use core_courseformat\output\section_renderer;
use moodle_page;

class renderer extends section_renderer {

    public function __construct(moodle_page $page, $target) {
        parent::__construct($page, $target);
        // Allow the "Turn editing on" link to appear for setcurrentsection capability.
        $page->set_other_editing_capability('moodle/course:setcurrentsection');
    }

    public function section_title($section, $course): string {
        return $this->render(course_get_format($course)->inplace_editable_render_section_name($section));
    }

    public function section_title_without_link($section, $course): string {
        return $this->render(course_get_format($course)->inplace_editable_render_section_name($section, false));
    }
}
```

### Content Output (classes/output/courseformat/content.php)

Override to inject AMD modules or change the content data exported to Mustache:

```php
<?php
namespace format_[formatname]\output\courseformat;

use core_courseformat\output\local\content as content_base;
use renderer_base;

class content extends content_base {

    /** @var bool Whether to show "add section" controls after each section. */
    protected $hasaddsection = true;

    public function export_for_template(renderer_base $output) {
        global $PAGE;
        // Require AMD modules for the reactive editor.
        $PAGE->requires->js_call_amd('format_[formatname]/mutations', 'init');
        $PAGE->requires->js_call_amd('format_[formatname]/section', 'init');
        return parent::export_for_template($output);
    }
}
```

### Section Output (classes/output/courseformat/content/section.php)

```php
<?php
namespace format_[formatname]\output\courseformat\content;

use core_courseformat\base as course_format;
use core_courseformat\output\local\content\section as section_base;
use stdClass;

class section extends section_base {

    protected $format;

    public function export_for_template(\renderer_base $output): stdClass {
        $data = parent::export_for_template($output);

        // Example: inject "add section" button after each section.
        if (!$this->format->get_sectionnum() && !$this->section->get_component_instance()) {
            $addsectionclass = $this->format->get_output_classname('content\\addsection');
            $addsection = new $addsectionclass($this->format, $this->section);
            $data->numsections = $addsection->export_for_template($output);
            $data->insertafter = true;
        }

        return $data;
    }
}
```

### Section Control Menu (classes/output/courseformat/content/section/controlmenu.php)

Extend to add custom actions (e.g. highlight/unhighlight):

```php
<?php
namespace format_[formatname]\output\courseformat\content\section;

use core\output\action_menu\link_secondary as action_menu_link_secondary;
use core\output\pix_icon;
use core_courseformat\output\local\content\section\controlmenu as controlmenu_base;
use core\url;

class controlmenu extends controlmenu_base {

    public function section_control_items(): array {
        $section = $this->section;
        $parentcontrols = parent::section_control_items();

        if ($section->is_orphan() || !$section->sectionnum) {
            return $parentcontrols;
        }

        if (!has_capability('moodle/course:setcurrentsection', $this->coursecontext)) {
            return $parentcontrols;
        }

        // Add a "Highlight" toggle after the Edit control.
        return $this->add_control_after($parentcontrols, 'edit', 'highlight', $this->get_highlight_item());
    }

    protected function get_highlight_item(): action_menu_link_secondary {
        $format  = $this->format;
        $section = $this->section;
        $course  = $format->get_course();

        $ison = ($course->marker == $section->sectionnum);

        $url = $this->format->get_update_url(
            action: $ison ? 'section_unhighlight' : 'section_highlight',
            ids: [$section->id],
            returnurl: $this->baseurl,
        );

        return new action_menu_link_secondary(
            url: $url,
            icon: new pix_icon($ison ? 'i/marked' : 'i/marker', ''),
            text: $ison ? get_string('highlightoff') : get_string('highlight'),
            attributes: [
                'class'            => 'editing_highlight',
                'data-action'      => $ison ? 'sectionUnhighlight' : 'sectionHighlight',
                'data-id'          => $section->id,
                'data-icon'        => $ison ? 'i/marked' : 'i/marker',
                'data-swapname'    => $ison ? get_string('highlight') : get_string('highlightoff'),
                'data-swapicon'    => $ison ? 'i/marker' : 'i/marked',
            ],
        );
    }
}
```

## State Actions (classes/courseformat/stateactions.php)

Handle custom AJAX mutations from the reactive editor:

```php
<?php
namespace format_[formatname]\courseformat;

use core_courseformat\stateupdates;
use core_courseformat\stateactions as stateactions_base;
use stdClass;
use context_course;

class stateactions extends stateactions_base {

    /**
     * Highlight a course section (custom action example).
     */
    public function section_highlight(
        stateupdates $updates,
        stdClass $course,
        array $ids = [],
        ?int $targetsectionid = null,
        ?int $targetcmid = null
    ): void {
        $this->validate_sections($course, $ids, __FUNCTION__);
        require_capability('moodle/course:setcurrentsection', context_course::instance($course->id));

        foreach ($ids as $sectionid) {
            $sectioninfo = $this->get_section_info($course->id, $sectionid);
            \core_courseformat\formatactions::section($course->id)->set_marker($sectioninfo, true);
            $updates->add_section_put($sectionid);
        }
        $updates->add_course_put();
    }
}
```

## Privacy Provider (classes/privacy/provider.php)

Course formats almost never store personal data — use the null provider:

```php
<?php
namespace format_[formatname]\privacy;

defined('MOODLE_INTERNAL') || die();

use core_privacy\local\metadata\null_provider;

class provider implements null_provider {
    public static function get_reason(): string {
        return 'privacy:metadata';
    }
}
```

## Backup/Restore (backup/moodle2/restore_format_[formatname]_plugin.class.php)

```php
<?php
defined('MOODLE_INTERNAL') || die();

class restore_format_[formatname]_plugin extends restore_format_plugin {

    /**
     * Returns a dummy path element so after_restore_course() is called.
     */
    public function define_course_plugin_structure(): array {
        return [new restore_path_element('dummy_course', $this->get_pathfor('/dummycourse'))];
    }

    public function process_dummy_course(): void {
        // Nothing to process.
    }

    public function after_restore_course(): void {
        // Post-restore logic — e.g. hide orphaned sections, migrate settings.
    }
}
```

## AMD Modules

### mutations.js (amd/src/mutations.js)

```javascript
import {getCurrentCourseEditor} from 'core_courseformat/courseeditor';
import DefaultMutations from 'core_courseformat/local/courseeditor/mutations';

class [Formatname]Mutations extends DefaultMutations {

    /**
     * Highlight sections — must be a class attribute (arrow function / field),
     * not a regular method, so plugin merging works correctly.
     *
     * @param {StateManager} stateManager
     * @param {number[]} sectionIds
     */
    sectionHighlight = async function(stateManager, sectionIds) {
        const course = stateManager.get('course');
        this.sectionLock(stateManager, sectionIds, true);
        const updates = await this._callEditWebservice('section_highlight', course.id, sectionIds);
        stateManager.processUpdates(updates);
        this.sectionLock(stateManager, sectionIds, false);
    };
}

export const init = () => {
    const editor = getCurrentCourseEditor();
    editor.addMutations(new [Formatname]Mutations());
};
```

### section.js (amd/src/section.js)

```javascript
import {BaseComponent} from 'core/reactive';
import {getCurrentCourseEditor} from 'core_courseformat/courseeditor';

class [Formatname]Section extends BaseComponent {

    create() {
        this.name = 'format_[formatname]_section';
        this.selectors = {
            HIGHLIGHT: '[data-action="sectionHighlight"]',
            UNHIGHLIGHT: '[data-action="sectionUnhighlight"]',
        };
    }

    getWatchers() {
        return [
            {watch: 'section.current:updated', handler: this._refreshHighlight},
        ];
    }

    async _refreshHighlight({element}) {
        // Update the highlight button state when section.current changes.
        const highlightBtn   = this.getElement(this.selectors.HIGHLIGHT);
        const unhighlightBtn = this.getElement(this.selectors.UNHIGHLIGHT);
        if (highlightBtn) {
            highlightBtn.classList.toggle('d-none', element.current);
        }
        if (unhighlightBtn) {
            unhighlightBtn.classList.toggle('d-none', !element.current);
        }
    }
}

export const init = () => {
    const editor = getCurrentCourseEditor();
    editor.getDispatcher().then((dispatcher) => {
        document.querySelectorAll('[data-format-section]').forEach((element) => {
            const component = new [Formatname]Section({element, reactive: editor});
            component.init();
        });
    });
};
```

## Version File (version.php)

```php
<?php
defined('MOODLE_INTERNAL') || die();

$plugin->version   = 2026010100;        // YYYYMMDDXX
$plugin->requires  = 2026041000;        // Minimum Moodle version (Moodle 5.x)
$plugin->component = 'format_[formatname]';
```

## Testing

### PHPUnit test skeleton

```php
<?php
defined('MOODLE_INTERNAL') || die();

class format_[formatname]_test extends advanced_testcase {

    public function setUp(): void {
        parent::setUp();
        $this->resetAfterTest();
    }

    public function test_format_options(): void {
        $course = $this->getDataGenerator()->create_course(
            ['format' => '[formatname]']
        );
        $format = course_get_format($course);
        $options = $format->get_format_options();
        $this->assertArrayHasKey('hiddensections', $options);
    }

    public function test_get_section_name(): void {
        $course = $this->getDataGenerator()->create_course(
            ['format' => '[formatname]', 'numsections' => 3]
        );
        $format = course_get_format($course);

        // Section 0 has a special name.
        $this->assertEquals(
            get_string('section0name', 'format_[formatname]'),
            $format->get_section_name(0)
        );
    }
}
```

## Deployment

```bash
# Deploy to Moodle (replace [formatname]):
rsync -a --delete /vagrant/PluginDev/moodle-format_[formatname]/ \
    /srv/lms/moodle/public/course/format/[formatname]/

# Trigger plugin upgrade:
php /srv/lms/moodle/admin/cli/upgrade.php --non-interactive
```

## Key Reference Implementations

Examine these installed formats for working patterns:

| Format | Path | Notable features |
|--------|------|-----------------|
| `topics` | `public/course/format/topics/` | Section highlighting, indentation, reactive mutations |
| `weeks`  | `public/course/format/weeks/`  | Date-based section names |
| `singleactivity` | `public/course/format/singleactivity/` | Minimal format, no sections |

## Common Issues

### 1. `format.php` executed directly → fatal error

`format.php` depends on globals set by `course/view.php` (`$course`, `$marker`, `$displaysection`). Never call it directly. If you need a standalone route, create a separate PHP file.

### 2. Missing `inplace_editable` callback → section names not editable

The free function `format_[formatname]_inplace_editable()` must exist in `lib.php` for in-place section name editing to work.

### 3. AMD build missing → JS mutations not registered

Run `grunt amd` (or `npx grunt amd`) in the Moodle root after editing `amd/src/*.js`. The `amd/build/` compiled files must be committed.

### 4. `stateactions` class not found → AJAX highlight fails

The class `format_[formatname]\courseformat\stateactions` must extend `core_courseformat\stateactions` and be autoloaded from `classes/courseformat/stateactions.php`.

### 5. Privacy provider missing → upgrade warning

Always add `classes/privacy/provider.php` implementing `null_provider`, even if the format stores no data.

### 6. `restore_format_*_plugin` class name mismatch → restore fails

The class name in the backup file must exactly match `restore_format_[formatname]_plugin` and the file must be at `backup/moodle2/restore_format_[formatname]_plugin.class.php`.
