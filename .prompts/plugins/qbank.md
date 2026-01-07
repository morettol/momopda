# Question Bank Plugin Development Guide - Moodle 4.5

You are developing a Moodle question bank (qbank) plugin for **Moodle 4.5+** with shared question banks. These plugins extend the question bank interface with additional columns, actions, filters, and controls.

## Plugin Architecture

### Core Components

1. **plugin_feature.php** - Main entry point extending `\core_question\local\bank\plugin_features_base`
2. **version.php** - Plugin metadata and version information
3. **Bulk action classes** - Extend `bulk_action_base` for multi-question operations
4. **Action classes** - Extend `question_action_base` for single question actions
5. **Column classes** - Extend `column_base` or `row_base` for data display
6. **Filter classes** - Extend filter base classes for search functionality

### Plugin Structure
```
qbank_yourplugin/
├── classes/
│   ├── plugin_feature.php              # Main feature registration
│   ├── [bulk_action_name]_action.php   # Bulk operations (multi-question)
│   ├── [action_name]_action.php        # Single question actions
│   ├── [column_name]_column.php        # Column displays
│   ├── [filter_name]_condition.php     # Search filters
│   ├── local/                          # Business logic classes
│   ├── task/                           # Background tasks
│   ├── privacy/provider.php            # Privacy API
│   └── output/
│       └── renderer.php                # Custom renderers
├── lang/en/
│   └── qbank_yourplugin.php            # Language strings
├── db/
│   ├── access.php                      # Capabilities
│   └── install.xml                     # Database schema
├── tests/
│   └── [test_files].php                # Unit tests
├── amd/src/                            # AMD modules (optional)
├── templates/                          # Mustache templates
├── [bulk_action_page].php              # Entry point for bulk actions
├── [action_page].php                   # Entry point for single actions
├── styles.css                          # Custom CSS
└── version.php
```

## Common Plugin Types & When to Use Each

### 🎯 Bulk Action Plugins (Multi-Question Operations)
- **Purpose**: Operations on multiple selected questions simultaneously
- **Examples**: Move questions, delete multiple questions, export selected questions, **bulk AI editing**
- **Integration**: "With selected" dropdown menu
- **Key Pattern**: Users select questions first, then choose bulk action
- **URL Receives**: `cmid` parameter + selected question IDs as `q[ID]` parameters
- **Moodle 4.5 Note**: Uses new question bank structure with versioning

### 🎯 Single Action Plugins (Individual Question Operations) 
- **Purpose**: Actions on one question at a time
- **Examples**: Edit question, preview question, duplicate question
- **Integration**: Action icons next to each question
- **Key Pattern**: Direct action on specific question
- **URL Receives**: Question ID parameter

### 🎯 Column Plugins (Data Display)
- **Purpose**: Display additional question data/metadata  
- **Examples**: Question text preview, usage statistics, custom fields
- **Integration**: Additional columns in question list table

### 🎯 Filter Plugins (Search Enhancement)
- **Purpose**: Enable advanced question searching/filtering
- **Examples**: Filter by question type, tags, usage, custom criteria
- **Integration**: Filter controls above question list

## 🚨 Critical Implementation Checklist

### Context Access
- [ ] ✅ Use `global $PAGE; $PAGE->context` in `plugin_feature.php`
- [ ] ❌ Never call `$this->get_question_bank()` in `plugin_feature.php`
- [ ] ✅ Handle context detection in target pages using `cmid` parameter
- [ ] ✅ Use `get_module_from_cmid()` and `context_module::instance()` pattern

### Bulk Actions
- [ ] ✅ Use `get_bulk_actions()` method for multi-question operations
- [ ] ❌ Never use navigation tabs for bulk operations
- [ ] ✅ Create simple static URLs in `get_bulk_action_url()`
- [ ] ✅ Extract question IDs using `q([0-9]+)` pattern in target page
- [ ] ✅ Verify permissions for each selected question individually
- [ ] ✅ Redirect back to question bank with `cmid` parameter using `/question/edit.php`

### Language Strings & Caching
- [ ] ✅ Use clear, descriptive string identifiers (`bulk_action_title`, not `bulk_ai_edit`)
- [ ] ✅ Increment version number when adding new strings
- [ ] ✅ Test string loading after plugin installation/upgrade
- [ ] ✅ Include all required strings in language file

### Security
- [ ] ✅ Check capabilities at both plugin and individual question level
- [ ] ✅ Validate and sanitize all input parameters
- [ ] ✅ Use `question_require_capability_on()` for individual questions
- [ ] ✅ Use parameterized database queries

## 🚨 COMPREHENSIVE TESTING CHECKLIST FOR MOODLE 4.5

### Database Compatibility
- [ ] ✅ Uses {question_versions} table in all queries
- [ ] ✅ No references to old q.category field
- [ ] ✅ Version filtering with MAX() subquery for latest versions
- [ ] ✅ Status filtering (excludes hidden questions)
- [ ] ✅ Complete question creation (all 3 tables: question, question_bank_entries, question_versions)
- [ ] ✅ Unique idnumber handling (no constraint violations)

### Context & Parameters Access
- [ ] ✅ Uses global $PAGE in plugin_feature.php (not get_question_bank())
- [ ] ✅ Target page handles cmid parameter correctly
- [ ] ✅ No calls to get_question_bank() methods
- [ ] ✅ Proper context derivation from cmid using get_module_from_cmid()
- [ ] ✅ Required files included (editlib.php for question functions)

### SQL & Parameters
- [ ] ✅ Consistent parameter naming (all named or all positional)
- [ ] ✅ No mixed parameter types in queries
- [ ] ✅ Proper parameter merging with array_merge()
- [ ] ✅ Use parameterized database queries (no SQL injection)

### URL Structure & Navigation
- [ ] ✅ Uses /question/edit.php (not /question/bank/view.php)
- [ ] ✅ Correct cmid-based URLs throughout
- [ ] ✅ No method chaining in array literals
- [ ] ✅ Proper redirect URLs after bulk actions

### Data Format Handling
- [ ] ✅ GIFT parser array format handled correctly (extract ['text'] field from answers/feedback)
- [ ] ✅ Feedback format preserved from parser
- [ ] ✅ Progress bar access via stored_progress_bar static methods

### User Interface & Integration
- [ ] ✅ Bulk actions appear in "With selected" dropdown
- [ ] ✅ No redundant question selection interfaces
- [ ] ✅ Clear action flow from selection to processing
- [ ] ✅ Consistent form processing pattern (AJAX vs traditional, not mixed)

### Capability & Security Testing
- [ ] ✅ Check capabilities at both plugin and individual question level
- [ ] ✅ Use `question_require_capability_on()` for individual questions
- [ ] ✅ Validate and sanitize all input parameters
- [ ] ✅ Test capability checking and permission scenarios

### Error Handling & Edge Cases
- [ ] ✅ Graceful handling of no questions selected (redirect with message)
- [ ] ✅ Proper redirects and error messages
- [ ] ✅ Database constraint violation prevention
- [ ] ✅ Test with multiple question types and large selections

### Language Strings & Installation
- [ ] ✅ Increment version number when adding new strings
- [ ] ✅ Include all required strings in language file
- [ ] ✅ Test plugin installation and language string loading

### Advanced Moodle 4.5 Features
- [ ] ✅ Test with Moodle 4.5 shared question bank structure
- [ ] ✅ Verify question creation includes all required tables
- [ ] ✅ Test question versioning compatibility
- [ ] ✅ Test with different question bank contexts (course, system)
- [ ] ✅ Verify background task processing and progress tracking

## Moodle 4.5 Database Structure

### Understanding Shared Question Banks

Moodle 4.5 uses a **three-table structure** for questions:

1. **{question}** - Contains question data (name, text, type, etc.)
2. **{question_versions}** - Links questions to question bank entries with versioning
3. **{question_bank_entries}** - Groups question versions in categories (enables sharing)
4. **{question_categories}** - Contains category and context information

### Required Join Pattern for All Queries

```sql
-- Standard pattern for getting questions with category info
FROM {question} q
JOIN {question_versions} qv ON qv.questionid = q.id
JOIN {question_bank_entries} qbe ON qbe.id = qv.questionbankentryid
JOIN {question_categories} qc ON qc.id = qbe.questioncategoryid

-- Always include version filtering
WHERE qv.status <> 'hidden'
AND qv.version = (SELECT MAX(v.version)
                  FROM {question_versions} v
                  WHERE v.questionbankentryid = qbe.id)
```

### Available Fields from Modern Structure

```php
// Question data
q.id, q.name, q.qtype, q.questiontext, q.generalfeedback

// Version data
qv.version, qv.status, qv.id as versionid

// Question bank entry
qbe.id as questionbankentryid, qbe.idnumber, qbe.ownerid

// Category data
qc.id as categoryid, qc.name as categoryname, qc.contextid
```

## Question Creation Checklist (Moodle 4.5)

For any plugin that creates questions, you MUST:

1. **Insert into question table** with proper fields
2. **Create question_bank_entries record** with unique idnumber
3. **Create question_versions record** linking them
4. **Handle question type-specific data** (multichoice, etc.)
5. **Validate idnumber uniqueness** in category before insertion

### Complete Question Creation Pattern

```php
// Create question record
$question->id = $DB->insert_record('question', $question);

// Create question bank entry with unique idnumber
$questionbankentry = new \stdClass();
$questionbankentry->questioncategoryid = $question->category;
$questionbankentry->idnumber = $this->generate_unique_idnumber($question);
$questionbankentry->ownerid = $question->createdby;
$questionbankentry->id = $DB->insert_record('question_bank_entries', $questionbankentry);

// Create question version
$questionversion = new \stdClass();
$questionversion->questionbankentryid = $questionbankentry->id;
$questionversion->questionid = $question->id;
$questionversion->version = 1;
$questionversion->status = \core_question\local\bank\question_version_status::QUESTION_STATUS_READY;
$questionversion->id = $DB->insert_record('question_versions', $questionversion);
```

## Updated URL Structure (Moodle 4.5)

- **Question Bank**: `/question/edit.php` (was `/question/bank/view.php`)
- **Question Banks Listing**: `/question/banks.php`
- **Always use `cmid` parameter** for question bank access


