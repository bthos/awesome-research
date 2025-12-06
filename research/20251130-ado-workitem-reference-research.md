<!-- markdownlint-disable-file -->

# Task Research Notes: ADO #Work Item Reference Implementation

## Research Executed

### File Analysis

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/vss-bundle-async-v-lcUSWia5oldBlKJrCYTEhUBtuEjVJoTJaSz66FClfc=`
  - Complete `TFS.Mention.WorkItems` module implementation
  - Contains `WorkItemMentionsRenderingProvider`, `WorkItemsMentionParser`, `WorkItemAutocompleteProvider`
  - CSS classes, URL generation, click handlers

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wiki-renderer-content.es6.cuMuSd.min.js.download`
  - Wiki renderer with `WorkItemMentionsRenderingProvider.registerMentionClickHandler`
  - Module loading via `ILegacyPlatformService.requireModules`

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wit-restclient-content.es6.K7rbpW.min.js.download`
  - Work Item REST API client
  - `getWorkItem`, `getWorkItems` API methods

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-work-web.new-work-item-form-markdown.es6.5r5xBb.min.js.download`
  - HTML to Markdown conversion with work item mention handling
  - Regex pattern: `/^#[0-9]+$/gi`

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wiki-renderer-content.min.css`
  - CSS styling for `mention-wi-link` class

### Code Search Results

- `WorkItemMentionsRenderingProvider`
  - Found in wiki renderer, registers click handlers
- `TFS.Mention.WorkItems` module path
  - Loaded dynamically: `Mention/Scripts/TFS.Mention.WorkItems`
- `mention-wi-link` CSS class
  - Hover styling: `text-decoration: underline`
- Work item regex pattern
  - `#[0-9]{1,10}` with word boundary separators

## Key Discoveries

### Module Architecture

ADO loads work item mention functionality via AMD modules:

```javascript
// Module loading in wiki renderer
e.requireModules([
  "Mention/Scripts/TFS.Mention.People",
  "Mention/Scripts/TFS.Mention.WorkItems"
])
```

### Regex Pattern for Parsing

The `WorkItemsMentionParser` uses this regex pattern:

```javascript
// Pattern to match #123456 format
var pattern = new RegExp(
  "(" + PATTERN_WORD_START_SEPARATOR + ")#([0-9]{1,10})(?=(" + PATTERN_WORD_END_SEPARATOR + "))",
  "ig"
);

// Where separators are defined as:
// PATTERN_WORD_START_SEPARATOR = "(?:^|[\\s\\[\\(])"  
// PATTERN_WORD_END_SEPARATOR = "(?:$|[\\s\\]\\)\\.,;:!?])"
```

Key characteristics:
- Requires word boundary before `#`
- Matches 1-10 digits after `#`
- Requires word boundary after number
- Case-insensitive, global matching

### CSS Classes

```css
/* Base mention link styling */
a.mention-link {
  text-decoration: none;
}

/* Work item specific link */
a.mention-wi-link:hover {
  text-decoration: underline;
}

/* Work item widget container */
.mention-widget-workitem,
.mention-widget-workitem-no-access {
  background: rgba(244,244,244,1);
  background: rgba(var(--palette-neutral-4,244, 244, 244),1);
  padding: 2px .333em;
  display: inline;
  word-break: break-word;
  margin: 0 1px;
  border-radius: 2px;
  line-height: 24px;
  font-style: normal;
  font-weight: normal;
}

/* Work item with colored border */
.mention-widget-workitem {
  border-left: 2px solid transparent;
}

/* Work item type ID styling */
.mention-widget-workitem-typeid,
.mention-widget-workitem .mention-widget-workitem-typeid {
  color: rgba(16,110,190,1);
  color: rgba(var(--palette-primary-shade-10,16, 110, 190),1);
}

/* Work item title styling */
.mention-widget-workitem-title {
  color: rgba(0,0,0,.9);
  color: var(--text-primary-color,rgba(0, 0, 0, .9));
  font-weight: 600; /* fontWeightSemiBold */
}

/* Work item state indicator */
.mention-widget-workitem-state {
  border-left: 1px solid;
  border-left-color: rgba(200,200,200,1);
  padding: 0 8px 0 4px;
  display: inline-flex;
  align-items: center;
  margin: 4px 2px 4px 8px;
  line-height: 14px;
  vertical-align: bottom;
}

/* State color circle */
.mention-widget-workitem-state .workitem-state-color {
  border: 3px solid transparent;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  margin: 0 4px;
  background-clip: padding-box;
}

/* Deleted/inaccessible work item */
.mention-widget-workitem-no-access {
  /* Same as .mention-widget-workitem but without border-left */
}
```

### HTML Output Structure

#### Simple Format (ID only)
```html
<a href="/{project}/_workitems/edit/{id}" 
   data-vss-mention="version:1.0"
   class="mention-link mention-wi-link">
  #123456
</a>
```

#### Rich Format (with title and state)
```html
<span class="mention-widget-workitem body-m" 
      style="border-left-color: {workItemTypeColor}">
  <a href="/{project}/_workitems/edit/{id}"
     class="mention-link mention-wi-link"
     data-wi="{serializedWorkItemData}"
     data-vss-mention="work-item"
     aria-label="{workItemType} {id}: {title}: {stateName}">
    <!-- Work item type icon -->
    <span class="work-item-type-icon-host">...</span>
    <!-- ID -->
    <span class="secondary-text">{id}</span>
    <!-- Title -->
    <span class="mention-widget-workitem-title fontWeightSemiBold">{title}</span>
  </a>
  <!-- State indicator -->
  <span class="mention-widget-workitem-state">
    <span class="workitem-state-color" style="background-color: {stateColor}"></span>
    <span>{stateName}</span>
  </span>
</span>
```

### Key Classes and Interfaces

#### CssClasses Enum
```typescript
enum CssClasses {
  MENTION_WIDGET_WORKITEM = "mention-widget-workitem",
  MENTION_WIDGET_WORKITEM_DELETED = "mention-widget-workitem-no-access"
}
```

#### WorkItemMentionsRenderingProvider
```typescript
class WorkItemMentionsRenderingProvider {
  static CSS_CLASS_WI_MENTION_LINK = "mention-wi-link";
  static CSS_CLASS_MENTION_CLICK_HANDLED = "mention-click-handled";
  static _WI_DATA_ATTRIBUTE = "data-wi";
  
  // Render a mention with full details
  renderMention(mention, wrapperFn, options) {
    return WorkItemProvider.getInstance()
      .getById(mention.ArtifactId)
      .then(workItem => {
        if (workItem === null) {
          // Work item not found or no access
          return createDeletedWorkItemElement(mention.ArtifactId);
        }
        return createWorkItemElement(workItem);
      });
  }
  
  // Register click handlers for work item links
  static registerMentionClickHandler(container, telemetryData, onClickCallback) {
    const links = $(".mention-wi-link:not(.mention-click-handled)", container);
    links.addClass("mention-click-handled");
    links.click(handleWorkItemClick);
  }
}
```

#### WorkItemsMentionParser
```typescript
class WorkItemsMentionParser extends ArtifactMentionParser {
  // Parse #123456 from plain text
  parseFromText(text) {
    const pattern = new RegExp(
      "(" + PATTERN_WORD_START_SEPARATOR + ")#([0-9]{1,10})(?=(" + PATTERN_WORD_END_SEPARATOR + "))",
      "ig"
    );
    const mentions = [];
    let match;
    while (match = pattern.exec(text)) {
      let start = match.index;
      let end = pattern.lastIndex;
      const prefix = match[1];
      if (prefix && prefix.length) {
        start += prefix.length;
      }
      mentions.push({
        index: { start, end },
        id: match[2]
      });
    }
    return mentions;
  }
  
  // Parse from URL like /_workitems/edit/123
  parseFromUrl(url) {
    const uri = new URI(url);
    const segments = uri.segment();
    // Skip to controller name "workitems"
    while (segments.shift() !== getControllerNameWithPrefix()) {
      if (!segments.length) return;
    }
    const id = +segments.pop();
    return id > 0 ? { workItemId: id } : undefined;
  }
  
  // Parse from HTML with data-vss-mention attribute
  parseFromHtml(container, callback) {
    const selector = 'a[href*="/_workitems/edit/"][data-vss-mention^="version:1.0"]';
    container.find(selector).each((i, el) => {
      const parsed = this.parseFromUrl($(el).attr("href"));
      if (parsed) callback($(el), parsed);
    });
  }
}
```

#### TfsMentionWorkItemHelpers
```typescript
namespace TfsMentionWorkItemHelpers {
  const EDIT_ACTION_NAME = "edit";
  const CONTROLLER_NAME = "workitems";
  
  // Generate work item URL
  function getWorkItemUrl(id: number, projectName?: string): string {
    const tfsContext = getMainTfsContext();
    return tfsContext.getPublicActionUrl(
      EDIT_ACTION_NAME, 
      CONTROLLER_NAME, 
      {
        parameters: id,
        project: projectName || tfsContext.navigation.projectId,
        team: ""
      }
    );
    // Result: /{project}/_workitems/edit/{id}
  }
  
  // Get controller name with prefix
  function getControllerNameWithPrefix(): string {
    const tfsContext = getMainTfsContext();
    return tfsContext.navigation.controllerPrefix + CONTROLLER_NAME;
    // Result: "_workitems"
  }
  
  // Navigate to work item
  function showWorkItem(id, workItem, url, event) {
    // Uses HubsService for SPA navigation
    const hubsService = getLocalService(HubsService);
    const handler = hubsService.getHubNavigateHandler("ms.vss-work-web.new-work-items-hub", url);
    handler(event);
  }
}
```

### Work Item REST API

```typescript
// API endpoint: {project}/_apis/wit/workItems/{id}
interface WorkItemTrackingClient {
  // Get single work item
  getWorkItem(
    id: number,
    project?: string,
    fields?: string[],
    asOf?: Date,
    expand?: WorkItemExpand
  ): Promise<WorkItem>;
  
  // Get multiple work items
  getWorkItems(
    ids: number[],
    project?: string,
    fields?: string[],
    asOf?: Date,
    expand?: WorkItemExpand,
    errorPolicy?: WorkItemErrorPolicy
  ): Promise<WorkItem[]>;
}

// API configuration
const WorkItemTrackingClientConfig = {
  apiVersion: "5.0-preview.3",
  routeTemplate: "{project}/_apis/wit/workItems/{id}",
  resourceAreaId: "5264459e-e5e0-4bd8-b118-0985e68a4ec5",
  serviceInstanceType: "00025394-6065-48ca-87d9-7f5672854ef7"
};
```

### Work Item Provider (Data Layer)

```typescript
class WorkItemProvider {
  private static _instance: WorkItemProvider;
  
  static getInstance(): WorkItemProvider {
    if (!this._instance) {
      this._instance = new WorkItemProvider();
    }
    return this._instance;
  }
  
  // Get work item by ID with caching
  getById(id: string): Promise<IWorkItem | null> {
    // Fetches from REST API, caches result
    // Returns null if work item doesn't exist or user has no access
  }
  
  // Search work items by ID prefix (for autocomplete)
  searchById(idPrefix: string): Promise<IWorkItem[]> {
    // Uses WIQL query to find matching work items
  }
  
  // Search work items by title/keyword
  search(query: string): Promise<IWorkItem[]> {
    // Full-text search via Work Item Search API
  }
}

interface IWorkItem {
  id: number;
  title: string;
  workItemType: string;
  projectName: string;
  colorAndIcon: {
    color: string;
    icon: string;
  };
  stateAndColor?: {
    stateName: string;
    stateColor: string;
  };
}
```

### HTML to Markdown Conversion

When converting HTML back to Markdown:

```typescript
function convertMentions(html: string): Promise<string> {
  const container = document.createElement("div");
  container.innerHTML = html;
  
  const mentionLinks = container.querySelectorAll("a[data-vss-mention]");
  
  for (const link of mentionLinks) {
    const innerHTML = link.innerHTML;
    
    // Check if it's a work item mention (#123)
    if (innerHTML.match(/^#[0-9]+$/gi)) {
      const id = innerHTML.substring(1);
      const textNode = document.createTextNode(`#${id}`);
      link.parentElement.insertBefore(textNode, link);
      link.remove();
    }
    
    // Check if it's a user mention (@<name>)
    if (innerHTML.match(/^@.+$/gi)) {
      const userId = link.getAttribute("data-vss-mention")?.split(",").pop();
      const textNode = document.createTextNode(`@<${userId}>`);
      link.parentElement.insertBefore(textNode, link);
      link.remove();
    }
  }
  
  return container.innerHTML;
}
```

### Autocomplete Integration

```typescript
class WorkItemAutocompleteProvider extends JQueryAutocompletePlugin {
  static MENU_WIDTH = 450;
  
  // Check if autocomplete should open
  canOpen(selection) {
    const text = this.getTruncatedText(selection);
    const match = this.mentionPattern().exec(text.textBeforeSelection);
    if (match) {
      return {
        start: match.index + match[1].length,
        end: selection.textBeforeSelection.length
      };
    }
    return null;
  }
  
  // Get work item suggestions
  getSuggestions(selection) {
    const numberMatch = NumberMentionPattern.exec(selection.textBeforeSelection);
    if (numberMatch) {
      // User typed #123, search by ID
      return this.getWorkItemProvider().searchById(numberMatch[2]);
    }
    // User typed #keyword, search by text
    const textMatch = this.mentionPattern().exec(selection.textBeforeSelection);
    return this.getWorkItemProvider().search(textMatch ? textMatch[2] : "");
  }
  
  // Render autocomplete suggestion item
  renderSuggestion(container, suggestion) {
    const item = $("<li/>").appendTo(container);
    const link = $("<a/>").appendTo(item);
    
    // Add work item type icon
    renderWorkItemTypeIcon(link[0], suggestion.workItemType, suggestion.colorAndIcon);
    
    // Add work item summary
    const summary = getWorkItemSummary(
      `<span class="mention-autocomp-id">${suggestion.workItemType} ${suggestion.id}</span>`,
      `<span class="mention-autocomp-title">${suggestion.title}</span>`
    );
    link.append(summary);
    
    return item;
  }
  
  // Get replacement text when item selected
  getReplacementText(selection, suggestion) {
    const text = this.getTruncatedText(selection);
    const match = this.mentionPattern().exec(text.textBeforeSelection);
    const insertPos = match.index + match[1].length + 1;
    
    let replacement = suggestion.id.toString();
    // Add space after if not followed by word boundary
    if (!AfterMentionPattern.exec(selection.textAfterSelection)) {
      replacement += " ";
    }
    
    return {
      textBeforeSelection: selection.textBeforeSelection.substr(0, insertPos) + replacement,
      textInSelection: "",
      textAfterSelection: selection.textAfterSelection
    };
  }
  
  // Get HTML replacement for rich editor
  getReplacementHtml(suggestion) {
    return createWorkItemHtmlMentionWithTitle(suggestion);
  }
}
```

### Helper Functions

```typescript
// Get work item mention text (simple format)
function getWorkItemMentionText(id: number): string {
  return `#${id}`;
}

// Get work item type with ID string
function getWorkItemTypeWithIdString(workItemType: string, id: number): string {
  return `${workItemType} ${id}`;
}

// Get work item summary (type + ID + title)
function getWorkItemSummary(typeIdHtml: string, titleHtml: string): string {
  return `${typeIdHtml}: ${titleHtml}`;
}

// Create HTML mention link
function createHtmlMention(href: string, displayText: string): string {
  return `<a href="${href}" class="mention-link mention-wi-link" data-vss-mention="work-item">${displayText}</a>`;
}

// Create work item HTML with ID only
function createWorkItemHtmlMentionWithId(workItem: IWorkItem): string {
  const url = getWorkItemUrl(workItem.id);
  const text = getWorkItemMentionText(workItem.id);
  return createHtmlMention(url, text);
}

// Create work item HTML with title
function createWorkItemHtmlMentionWithTitle(workItem: IWorkItem): string {
  const url = getWorkItemUrl(workItem.id);
  const summary = getWorkItemSummary(
    getWorkItemTypeWithIdString(workItem.workItemType, workItem.id),
    workItem.title
  );
  return createHtmlMention(url, summary);
}
```

## Recommended Approach

For the Milkdown editor implementation:

### 1. Parsing (Markdown → Editor)

Create a remark plugin that:
- Matches `#[0-9]{2,}` pattern (require 2+ digits per project convention)
- Converts to a custom `workItemRef` node type
- Preserves the numeric ID

### 2. Schema (ProseMirror)

Define a mark or inline node:
```typescript
const workItemRefMark = {
  attrs: {
    id: { default: null }
  },
  inclusive: false,
  parseDOM: [{
    tag: "a.mention-wi-link",
    getAttrs: (dom) => ({
      id: dom.getAttribute("data-wi-id") || dom.textContent?.match(/#(\d+)/)?.[1]
    })
  }],
  toDOM: (mark) => [
    "a",
    {
      class: "mention-link mention-wi-link",
      href: `#${mark.attrs.id}`,
      "data-wi-id": mark.attrs.id
    },
    `#${mark.attrs.id}`
  ]
};
```

### 3. Serialization (Editor → Markdown)

Output plain `#123456` text - no special HTML wrapping needed since ADO markdown supports this natively.

### 4. Styling

Apply CSS from ADO's official implementation:
```css
a.mention-wi-link {
  text-decoration: none;
  color: var(--communication-foreground, rgba(0, 90, 158, 1));
}

a.mention-wi-link:hover {
  text-decoration: underline;
}
```

## Implementation Guidance

- **Objectives**: Parse `#123456` work item references and render as styled links
- **Key Tasks**:
  1. Create remark plugin to parse `#\d{2,}` pattern
  2. Define ProseMirror mark schema for work item references
  3. Add CSS styling matching ADO's official styles
  4. Ensure serialization outputs plain `#123456` text
- **Dependencies**: None (self-contained, no API calls needed for basic implementation)
- **Success Criteria**:
  - `#123` renders as a styled link in the editor
  - Editing the number updates the reference
  - Serialized markdown contains `#123` text
  - Visual styling matches ADO's wiki appearance
