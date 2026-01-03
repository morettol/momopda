# Moodle Design Principles & Component Library Reference

This document provides a condensed reference for implementing UI features that align with Moodle's design standards and component library. Use this as your starting point, then access specific component pages as needed.

## Quick Reference: Core Principles

**Central Purpose:** The Moodle Component Library documents frequently used UI components to enable efficient, consistent interface development across the platform.

**Key Philosophy:** Developers should leverage existing documented components rather than creating custom solutions. This reduces redundancy, improves consistency, and enhances user experience.

**Design Foundation:**
- Bootstrap 4 components are included with Moodle 4.5 installation.
- All components represent approved, safe building blocks for frontend development

**Bootstrap 4/5 Hybrid Support (Moodle 4.5):**
- Bootstrap 4.6 remains the primary version
- Bootstrap 5 syntax fully supported via bridge layer
- Class name migration: BOTH `.ml-*` AND `.ms-*` work (BS4 + BS5 coexistence)
- Data attributes: BOTH `data-toggle` AND `data-bs-toggle` supported
- jQuery 3.6.x is required in Moodle 4.5
- Form classes: BOTH `.custom-select` AND `.form-select` work simultaneously
- **Recommended:** Use Bootstrap 5 syntax for new development

**Target Audience:** Designers, developers, and UX professionals creating core Moodle code or extensions.

---

## **Moodle 4.5 Bootstrap Migration Status**

**⚠️ Important:** Moodle 4.5 is in a **transitional state** between Bootstrap 4 and Bootstrap 5. Understanding this hybrid state is critical for plugin development.

### **Current State: Steps 1-4 Completed**

Moodle 4.5 has completed the first four steps of the Bootstrap 5 migration (steps 5-7 are completed in Moodle 5.0):

#### **✅ Step 1: PopperJS Upgrade (v1 → v2)**
- **Status:** COMPLETED in Moodle 4.5
- **Impact:** Tooltip and dropdown positioning library upgraded
- **Developer Action:** 
  - Use PopperJS v2 API if customizing positioning
  - Legacy PopperJS v1 syntax no longer supported
  - Documentation: https://popper.js.org/docs/v2/migration-guide/

#### **✅ Step 2: SCSS Deprecation Process**
- **Status:** ACTIVE in Moodle 4.5
- **Impact:** Bootstrap 4 SCSS features marked for removal
- **Deprecated Functions:**
  - `theme-color-level()` → Use `shift-color()` instead
  - `color-yiq()` → Use custom contrast functions
  - String-based color functions deprecated
- **Developer Action:**
  - Review deprecation warnings in SCSS compilation
  - Update custom theme SCSS to avoid deprecated functions
  - Test themes with `--strict` mode enabled

#### **✅ Step 3: Refactoring BS4 Features Dropped in BS5**
- **Status:** COMPLETED in Moodle 4.5
- **Refactored Components:**
  - **Badges:** `.badge-*` classes retain BS4 behavior, but BS5 `.text-bg-*` equivalents available
  - **Forms:** `.custom-select`, `.custom-check` remain functional alongside BS5 alternatives
  - **Utilities:** `.sr-only` works alongside `.visually-hidden`
  - **Spacing:** Both `.ml-*` (BS4) and `.ms-*` (BS5) supported
- **Developer Action:**
  - **New development:** Use Bootstrap 5 syntax where possible
  - **Legacy code:** Bootstrap 4 syntax remains fully functional
  - Both syntaxes work simultaneously during transition

#### **✅ Step 4: Bootstrap 5 Bridge Layer**
- **Status:** ACTIVE in Moodle 4.5
- **Purpose:** Maintains backwards compatibility while enabling BS5 adoption
- **Bridge Features:**
  - Class name aliasing (`.ml-2` automatically maps to `.ms-2` internally)
  - Data attribute handling (`data-toggle` works alongside `data-bs-toggle`)
  - jQuery compatibility layer for Bootstrap components
  - SCSS variable mapping between BS4 and BS5 naming
- **Developer Action:**
  - **Recommended:** Write new code using Bootstrap 5 syntax
  - **Supported:** Bootstrap 4 syntax remains fully compatible
  - **Benefit:** Forward compatibility with Moodle 5.0+

### **🎯 Development Recommendations for Moodle 4.5**

| Scenario | Recommendation |
|----------|----------------|
| **New Plugin Development** | Use Bootstrap 5 syntax (`.ms-*`, `.visually-hidden`, `.form-select`) for forward compatibility |
| **Updating Existing Plugin** | Both BS4 and BS5 syntax work; prioritize BS5 for new features |
| **Custom Theme SCSS** | Audit for deprecated SCSS functions; use `shift-color()` instead of `theme-color-level()` |
| **JavaScript Components** | Use `data-bs-*` attributes; jQuery bridge layer handles compatibility |
| **Testing** | Test with both BS4 and BS5 class names to ensure bridge layer works |

### **⏭️ Moodle 5.0 Changes (Steps 5-7)**

**NOT yet in Moodle 4.5** (available in Moodle 5.0+):
- Step 5: Bootstrap 5 becomes the default
- Step 6: Bootstrap 4 compatibility layer deprecation warnings
- Step 7: Bootstrap 4 bridge removal (Moodle 6.0+)

For detailed Bootstrap 5 migration guidance (class name changes, data attributes, SCSS functions, etc.), consult:
**https://moodledev.io/docs/4.5/guides/bs5migration**

Use this page if you encounter:
- Deprecated Bootstrap 4 class names in existing code
- Data attribute compatibility issues
- SCSS function updates needed (e.g., `theme-color-level()` → `shift-color()`)
- Form or badge component upgrades

### **📚 Migration Resources**

- **Official BS5 Migration Guide:** https://moodledev.io/docs/4.5/guides/bs5migration
- **PopperJS v2 Migration:** https://popper.js.org/docs/v2/migration-guide/
- **Bootstrap 5 Documentation:** https://getbootstrap.com/docs/5.3/
- **SCSS Deprecations:** https://moodledev.io/general/development/policies/deprecation/scss-deprecation

---

## Complete Navigation Index

### Moodle Components (UI Building Blocks)
Each component has comprehensive documentation on usage, examples, and implementation details.

| Component | URL |
|-----------|-----|
| Activity Icons | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/activity-icons/ |
| Buttons | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/buttons/ |
| Course Cards | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/course-cards/ |
| Form Elements | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/form-elements/ |
| Footer | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/footer/ |
| Icons | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/icons/ |
| Notification Badges | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/notification-badges/ |
| Notifications | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/notifications/ |
| Toggle Input | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/toggle-input/ |
| Search Input | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/search-input/ |
| Show More | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/show-more/ |
| Collapsible Sections | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/collapsable-sections/ |
| Task Indicator | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/task-indicator/ |
| Action Menus | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/action-menus/ |
| Dropdowns | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/dropdowns/ |
| HTML Modals | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/html-modals/ |
| Dynamic Tabs | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/components/dynamic-tabs/ |

### JavaScript Functionality
Interactive utilities and data visualization components.

| Feature | URL |
|---------|-----|
| Confirm Dialog | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/javascript/confirm/ |
| Toast Notifications | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/javascript/toast/ |
| Emoji Picker | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/javascript/emojipicker/ |
| Sortable Lists | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/javascript/sortable-list/ |
| Moodle Charts | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/javascript/charts/ |

### Design Standards & Themes
Guidelines for colors, spacing, typography, layout, and visual hierarchy.

| Standard | URL |
|----------|-----|
| Colours | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/themes/colours/ |
| Grids | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/themes/grids/ |
| Positioning | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/themes/positioning/ |
| Spacing | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/themes/spacing/ |
| Icon Sizes | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/themes/icon-sizes/ |
| Layout | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/themes/layout/ |
| Text | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/themes/text/ |

### Accessibility Guidelines
Standards for accessible UI implementation.

| Guideline | URL |
|-----------|-----|
| Links | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/accessibility/links/ |
| Colour Contrast | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/accessibility/colour-contrast/ |
| Keyboard Access | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/moodle/accessibility/keyboard-access/ |

### Component Library Documentation
Internal reference for contributing to the library.

| Topic | URL |
|-------|-----|
| Getting Started | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/library/getting-started/ |
| Adding Pages | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/library/adding-pages/ |
| Adding Images | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/library/adding-images/ |
| Syntax Highlighting | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/library/syntax-highlighting/ |
| Component Library Backend | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/library/component-library-backend/ |
| Example Files | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/library/example-files/ |
| Moodle JavaScript | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/library/moodle-javascript/ |
| Moodle Templates | https://componentlibrary.moodle.com/admin/tool/componentlibrary/docspage.php/library/moodle-templates/ |

---

## Key Design Guidelines (Summary)

### Components to Use
When implementing UI features, prioritize these pre-built, tested components:

**Interactive Elements:** Buttons, toggles, dropdowns, modals, tabs, action menus
**Form Structures:** Standardized form inputs and elements
**Visual Indicators:** Badges, notifications, task indicators
**Navigation:** Action menus, collapsible sections
**Information Display:** Cards, footers, icons

### Design Standards to Follow

**Bootstrap Foundation:**
- All components are built on Bootstrap, which is included with every Moodle installation
- Refer to official Bootstrap documentation for additional component details
- Use Bootstrap utilities for layout, spacing, and responsive design

**Accessibility:**
- Ensure proper color contrast for readability
- Support keyboard navigation for all interactive elements
- Use semantic HTML and ARIA labels where appropriate
- Test components for accessibility compliance

**Visual Consistency:**
- Follow Moodle's color palette (see Colours standard)
- Use consistent spacing and grids (see Spacing & Grids standards)
- Maintain typographic hierarchy (see Text standard)
- Use consistent icon sizing (see Icon Sizes standard)
- Follow layout patterns (see Layout standard)

---

## For Agent Implementation

When implementing UI features for Moodle:

1. **Start here:** Review this document for core principles
2. **Check for existing components:** Search the Components section above
3. **Review design standards:** Consult the Themes section for visual guidelines
4. **Verify accessibility:** Check the Accessibility Guidelines section
5. **Access detailed docs:** Visit the component page URL for full documentation, examples, and code samples
6. **Use Bootstrap:** Leverage Bootstrap components as the foundation for any custom elements

---

## Additional Resources

- **Official Moodle Component Library:** https://componentlibrary.moodle.com/
- **Moodle Developer Documentation:** https://moodledev.io/docs/4.5

---

*This reference was created as a condensed guide for efficient UI development aligned with Moodle standards. For comprehensive examples, implementation details, and component previews, visit the full Moodle Component Library at the URLs listed above.*
