# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

```bash
# Install dependencies
npm install

# Development server (port 9698)
npm run serve

# Production build
npm run build

# Lint and fix files
npm run lint

# Start JSON mock server (for development)
npm run json:server
```

## Architecture Overview

This is a Vue 2.x micro-frontend application using Module Federation for the **Page Builder** system. The application allows users to create dynamic pages with SQL data, HTML templates, and custom filters.

### Core Structure

- **Entry Point**: `src/main.js` → `src/bootstrap.js` (micro-frontend pattern)
- **Config Loading**: Dynamic configuration via `config.js` alias resolving to `./config/`
- **Router**: Vue Router with lazy loading, main routes: `/Builder`, `/Viewer`
- **State**: Vuex store with modular architecture
- **Styling**: Bootstrap 4 + Bootstrap Vue + SCSS

### Key Modules

#### Builder Module (`src/modules/Builder/`)
- **Builder.vue**: Main page creation interface
- **Components**: Editor, FileUpload, Header, PagePreview, SideBar, Stage
- Purpose: Create and edit pages with SQL scripts, HTML templates, and filters

#### Viewer Module (`src/modules/Viewer/`)
- **Viewer.vue**: Standard page viewing
- **StandaloneViewer.vue**: Embeddable page viewing (`/viewer-page/:pageId`)
- **IndependentPageDisplay.vue**: Core page rendering logic

#### Code Assistance (`src/modules/CodeAssistanceArchive/`)
- **Snippets**: Pre-built components (Cards, Charts, Grids, Tables, etc.)
- **Help**: Video tutorials and documentation
- **Utility**: Helper functions and tools

### Technical Stack

- **Frontend**: Vue 2.6.11 + Vue Router + Vuex
- **UI Framework**: Bootstrap 4 + Bootstrap Vue
- **Charts**: Chart.js + vue-chartjs + TreeMap support
- **Code Editor**: Monaco Editor (JS, CSS, HTML, TS, JSON)
- **Math Rendering**: MathJax (bundled in `/MathJax/`)
- **Pivot Tables**: vue-pivottable
- **Template Engine**: Nunjucks-based templating
- **Module Federation**: Webpack 5 federation for micro-frontend architecture

### API Layer

- **credCAPI**: Custom API wrapper (`src/utils/credCAPI/`)
- **Endpoints**: RESTful APIs for pages and sections (CRUD operations)
- **Mock Server**: JSON server for development (`automating.tools/json-server/`)
- **Config-driven**: API base URL determined by `config.jsonserver` flag

### Store Modules

- **pages.store.js**: Page management state
- **fileSystem.store.js**: File operations
- **notifications.store.js**: User notifications
- Global state: loading, theme, reference counting

### Development Features

- **Hot Reload**: Vue CLI dev server on port 9698
- **Source Maps**: Enabled for debugging
- **MathJax**: Auto-copied to dev server during development
- **Monaco**: Webpack plugin for code editor
- **Package Federation**: Exposes Chart.js and PivotTable components

### Configuration

- **Vue Config**: `vue.config.js` with micro-frontend setup
- **Babel**: Standard Vue preset configuration
- **Environment**: Config loaded dynamically, supports multiple deployment environments

### Page Builder Workflow

1. **Page Creation**: SQL script + HTML template + optional filters
2. **Section Management**: Reusable components within pages  
3. **Template System**: Nunjucks templating with Bootstrap components
4. **Data Binding**: SQL_DATA exposed to templates
5. **Viewer Rendering**: Dynamic page rendering with filter support

### Special Notes

- Uses custom `@credenceanalytics` packages for micro-frontend utilities
- Monaco editor configured for web development languages
- MathJax integration for mathematical expressions
- Supports embeddable page viewing via iframe/object tags
- Template engine supports Bootstrap components and custom snippets

### Attribute Handler System

Page Builder uses special HTML attributes for functionality without JavaScript:

**Section Embedding**: `data-pb-section` with sub-attributes:
- `data-pb-section-trigger-on-load` - Auto-load sections
- `data-pb-section-target` - Target containers
- `data-pb-section-params` - Pass parameters
- `data-pb-section-toggle` - Toggle visibility
- `data-pb-section-hide-header` - Hide section headers
- `data-pb-tab-name` - Custom tab names

**Tab Management System**:
- `activeTabSections` - Reactive object tracking active tabs per target
- `data-pb-active` - Reactive attribute showing active tab state
- Available in templates: `:data-pb-active="activeTabSections['.target'] === 'tab-name'"`
- Automatic state management across sections and pages

**Navigation**: 
- `data-pb-deeplink` (external links) with `data-pb-deeplink-params` and `data-pb-deeplink-target`
- `data-pb-link` (internal links) with `data-pb-link-params`

**Actions**:
- `data-pb-refresh` - Refresh page/section
- `data-pb-goto-home-page` - Navigate to home

**Embedding Patterns**:
- Sections within Pages
- Sections within Sections  
- Cross-section communication
- Dynamic parameter passing

### Section Development Requirements

When creating modular pages with sections:

**Directory Structure**:
```
page_name/
├── config.json                    # Page configuration with sections
├── pagetemplate.njk               # Main page template
├── pagescript.js                  # Main page script
├── section_name/
│   ├── pagesection-template.njk   # Section template
│   └── pagesection-script.js      # Section script
```

**Config.json Format**:
```json
{
  "pageid": "page_name",
  "pagename": "Page Display Name",
  "pagesections": [
    {
      "PAGESECTIONID": "section_name",
      "PAGESECTIONNAME": "SECTION DISPLAY NAME"
    }
  ],
  "is_system": "N"
}
```

**Database Registration** (Oracle):
```sql
Insert into AB_PAGESECTIONS (PAGEID,PAGESECTIONID,PAGESECTIONNAME,CREATED_BY,CREATED_ON,LASTUPDATED_BY,LASTUPDATED_ON) 
values ('page_name','section_name','SECTION NAME','system',SYSDATE,'system',SYSDATE);
```

**Live Interactivity Implementation**:
- Use `page3.showSection(sectionName, target, options)` for dynamic loading
- Implement `page3.calendar` or similar namespace for custom functions
- Use `page3.setSessionProcess(sectionName, data)` for cross-section communication
- Handle section loading with `data-pb-section` attributes

**Session Process Communication**:
```javascript
// Store data for specific section
page3.setSessionProcess("SECTION NAME", { key: "value" });

// Section script receives data via session parameter
module.exports = {
    load: async function ({ filters, userobj, session }) {
        const sessionData = session || {};
        const value = sessionData.key; // Access stored data
    }
}
```

**Session Process API Integration**:
- Pages: `session_process[pageId].getRaw()` sent to `/pagebuilder/page/process/${pageId}`
- Sections: `session_process[sectionName].getRaw()` sent to `/pagebuilder/pagesection/process/${sectionId}`
- First parameter of `setSessionProcess` must be section name (`PAGESECTIONNAME`) or pageId

**Example Implementation**: See `calender` page for complete modular calendar system with:
- 4 interactive sections (header, display, details, filters)
- Live Page Builder attribute handlers
- Session-based cross-section communication
- State management without page refreshes

### Section-Scoped JavaScript Architecture

**CRITICAL: Section-Scoped Module Exports**

Each Page Builder section's `module.export` functions are **scoped to that specific section only**. Functions defined in one section's template cannot be called from other sections.

#### Correct Architecture Pattern

**1. Server-Side Scripts (pagesection-script.js):**
```javascript
// ONLY for server-side data processing
module.exports = {
    load: async function ({ filters, userobj, session }) {
        // Handle context preparation, database queries, session data processing
        // Should NOT contain client-side JavaScript functions
        const sessionData = session || {};
        return { data: "processed server-side data" };
    }
}
```

**2. Client-Side Functions (in template `<script>` tags):**
```html
<!-- Section template content -->
<div>...</div>

<script>
module.export = {
    functionName: () => {
        // Client-side JavaScript function
        // Can call page3.setSessionProcess() for cross-section communication
        // Can call page3.showSection() to reload other sections
    }
};
</script>
```

#### Function Naming Convention

Functions are accessible using the section name as namespace:
- Section "CALENDAR HEADER" → `page3.calendar_header.functionName()`
- Section "EVENT DETAILS" → `page3.event_details.functionName()`  
- Section "CALENDAR DISPLAY" → `page3.calendar_display.functionName()`

**Example Implementation:**
```html
<button onclick="page3.calendar_header.handleViewChange('month')"
        data-pb-section="CALENDAR DISPLAY"
        data-pb-section-target="#calendar-display-container">
    Month View
</button>

<script>
module.export = {
    handleViewChange: (view) => {
        // Process view change
        page3.setSessionProcess("CALENDAR DISPLAY", { view: view });
        // Attribute handler will reload target section automatically
    }
};
</script>
```

#### Cross-Section Communication Pattern

**Proper Communication Flow:**
1. Function in Section A calls `page3.setSessionProcess("SECTION B", data)`
2. Section A uses `data-pb-section` attribute to reload Section B
3. Section B's pagesection-script.js receives the data in `session` parameter
4. Section B processes the data and renders accordingly

**Combined Attribute Handler Pattern:**
```html
<!-- Combines onclick function with section reload -->
<button onclick="page3.section_name.function()"
        data-pb-section="TARGET SECTION"
        data-pb-section-target="#target-container">
    Interactive Button
</button>
```

#### Common Errors and Solutions

**ERROR**: `Cannot read properties of undefined (reading 'functionName')`
**CAUSE**: Trying to call function from wrong section scope
**SOLUTION**: Move function to correct section template or use cross-section communication

**Before (INCORRECT):**
```html
<!-- In Section A template -->
<button onclick="page3.calendar.handleDayClick()">Click</button>
<!-- Function is actually in Section B -->
```

**After (CORRECT):**
```html
<!-- Option 1: Move function to Section A -->
<button onclick="page3.section_a.handleDayClick()">Click</button>

<!-- Option 2: Use cross-section communication -->
<button onclick="page3.section_a.triggerDayClick()"
        data-pb-section="SECTION B"
        data-pb-section-target="#section-b-container">Click</button>
```

#### Best Practices

1. **Keep functions in their respective section templates**
2. **Use descriptive section-based namespaces**
3. **Combine onclick handlers with data-pb-section attributes for live updates**
4. **Use `page3.setSessionProcess()` for cross-section data sharing**
5. **Server-side scripts should only handle data processing, not UI functions**

### API Calls with credCAPI in Templates

Templates can make REST API calls using `this.$credCAPI`:

**Basic Pattern**:
```javascript
this.$credCAPI
    .collection()
    .read({ 
        body: {
            "dataobj": {
                "event": "template_load",
                "Template": "portfolio_summary"
            }
        },
        requestURL: `/api/templates/load` 
    })
    .then((response) => {
        if (response && response.status == "unsuccess") {
            return console.error(response.msg || response.error);
        }
        // Process successful response
    });
```

**Key Points**:
- Use `this.$credCAPI.collection().method()` for API calls
- Structure request data in `body.dataobj`
- Specify complete `requestURL` for each endpoint
- Check `response.status == "unsuccess"` for errors
- credCAPI handles authentication automatically