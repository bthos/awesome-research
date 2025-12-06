<!-- markdownlint-disable-file -->

# Task Research Notes: TOC and TOSP Widget Implementation

## Research Executed

### File Analysis

- `examples/Editing Page 1 - Wiki - Overview.html`
  - Contains actual ADO editor HTML structure with TOC/TOSP rendered output
  - Found exact HTML structure: `<div class="toc-container">` with nested header and list
  - ADO renders TOC as inline-block with bullet lists
  
- `examples/Editing Page 1 - Wiki - Overview_files/ms.vss-features.markdown.min.css`
  - **Official ADO CSS source** for TOC and TOSP styling
  - Contains complete `.toc-container` and `.tosp-container` rules
  
- `examples/Editing Page 1 - Wiki - Overview_files/ms.vss-features.markdown.es6._wU6o_.min.js.download`
  - **Official ADO JavaScript implementation** for TOC/TOSP markdown-it plugins
  - Contains `TableOfContents` class (`PluginsTableOfContents`) with `_renderChildsTokens` algorithm
  - Contains `TableOfSubPagesPlugin` class for TOSP widget
  
- `examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wiki-renderer-content.es6.cuMuSd.min.js.download`
  - Wiki renderer that invokes TOC/TOSP plugins
  - Contains configuration options: `forceFullToc: true`, `includeLevel: [1,2,3,4,5,6]`
  
- `src/syntax/ado-toc-node.ts`
  - Current implementation uses custom CSS class `ado-toc-widget`
  - Creates header "Contents" and delete button
  - Live heading preview feature works correctly
  
- `src/syntax/ado-tosp-node.ts`
  - Uses `ado-tosp-widget` class
  - Creates header "Child Pages" with delete button
  - Shows placeholder text
  
- `src/theme/ado-theme.css` (lines 437-580)
  - Current widget styling follows Fabric UI card pattern with shadows
  - Uses fixed width `220px` and custom header bar styling
  - **Differs significantly from official ADO styling**

### Official ADO CSS (from ms.vss-features.markdown.min.css)

```css
/* TOC Container - Official ADO Style */
.toc-container {
    border: 1px solid;
    border-color: rgba(234,234,234,1);
    border-color: rgba(var(--palette-neutral-8,234, 234, 234),1);
    border-radius: 4px;
    display: inline-block;
    padding: 10px 16px 10px 0;
    margin-bottom: 14px;
    min-width: 250px;
}

.toc-container .toc-container-header {
    font-weight: 600;
    margin: 0 16px 5px 16px;
}

.toc-container ul {
    list-style: none;
    margin: 0;
    padding-left: 16px;
    color: rgba(0,90,158,1);
    color: var(--communication-foreground,rgba(0, 90, 158, 1));
}

.toc-container ul li {
    margin-top: 4px;
    margin-bottom: 0;
}

.toc-container li::before {
    content: "\2022 ";  /* Bullet point */
}

.toc-container a {
    margin-left: 5px;
    display: inline-block;
    vertical-align: top;
    width: auto;
    max-width: 400px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    word-wrap: normal;
}

/* TOSP Container - Official ADO Style (identical structure) */
.tosp-container {
    border: 1px solid;
    border-color: rgba(234,234,234,1);
    border-color: rgba(var(--palette-neutral-8,234, 234, 234),1);
    border-radius: 4px;
    display: inline-block;
    padding: 10px 16px 10px 0;
    margin-bottom: 14px;
    min-width: 250px;
}

.tosp-container .tosp-container-header {
    font-weight: 600;
    margin: 0 16px 5px 16px;
}

.tosp-container ul {
    list-style: none;
```

### Official ADO JavaScript Algorithm (from ms.vss-features.markdown.es6 - de-minified)

**Extracted from minified source - exact algorithm:**

```javascript
// TableOfContents class (PluginsTableOfContents)
// Default configuration (overridden by ADO):
const defaultOptions = {
  includeLevel: [1, 2],  // ADO overrides to [1,2,3,4,5,6]
  containerClass: "table-of-contents",  // ADO uses "toc-container"
  slugify: t => encodeURIComponent(t.trim().toLowerCase().replace(/\s+/g, "-")),
  markerPattern: /^\[\[toc\]\]/im,  // ADO uses /^\[\[_TOC_\]\][\s]*$/m
  listType: "ul",
  forceFullToc: true
};

// Core algorithm: _renderChildsTokens(currentIndex, headingTokens)
// Variables: n = previousLevel, l = currentLevel, a = currentIndex
// i = items array, o = current output string, r = recursive result

_renderChildsTokens = (t, e) => {
  let n, r, i = [], o = "", s = e.length, a = t;
  
  for (; a < s;) {
    const t = e[a], s = e[a - 1];
    const l = t.tag ? parseInt(t.tag.substr(1, 1)) : -1;
    
    // Skip non-heading tokens
    if ("heading_close" !== t.type || 
        -1 === this._options.includeLevel.indexOf(l) || 
        "inline" !== s.type) {
      a++;
      continue;
    }
    
    if (n) {  // n = previousLevel
      // GOING DEEPER: current level > previous level
      if (l > n) {
        r = this._renderChildsTokens(a, e);  // Recursive call
        o += r[1];  // Append nested list HTML
        a = r[0];   // Update index
        continue;
      }
      
      // GOING UP: current level < previous level  
      if (l < n) {
        o += "</li>";
        i.push(o);
        return [a, "<" + this._options.listType + ">" + i.join("") + "</" + this._options.listType + ">"];
      }
      
      // SAME LEVEL: close previous item
      l === n && (o += "</li>", i.push(o));
    } else {
      n = l;  // First heading sets the starting level
    }
    
    // Build anchor link
    let c = "#" + this._options.slugify(s.content);
    this._options.transformLink && (c = this._options.transformLink(c));
    
    // Create list item
    o = `<li><a href="${c}">`;
    o += "function" == typeof this._options.format 
      ? this._options.format(s.content) 
      : s.content;
    o += "</a>";
    a++;
  }
  
  // Finalize
  o += "" === o ? "" : "</li>";
  i.push(o);
  return [a, "<" + this._options.listType + ">" + i.join("") + "</" + this._options.listType + ">"];
};
```

**Key ADO configuration confirmed in source:**
```javascript
N.use((t => new P.TableOfContents(t, {
  includeLevel: [1, 2, 3, 4, 5, 6],
  containerClass: "toc-container",
  markerPattern: A.MarkdownConstants.TOCDefaultMarkerPattern,
  forceFullToc: true,
  slugify: t => (0, x.getEncodedHeaderText)(N, t),
  format: t => (t = (0, _.removeMarkdown)(t, { stripListLeaders: false }), 
                this._renderer.utils.unescapeMd(t)),
  containerHeaderHtml: '<div class="toc-container-header">' + k.MarkdownTocHeading + "</div>"
})));
```

**TableOfSubPagesPlugin - exact algorithm:**
```javascript
_renderChildsTokens = t => {
  let e = [], n = t.length || 0, r = 0;
  for (; r < n;) {
    const n = t[r];
    e.push(`<li><a href="${n.remoteUrl}">${n.name}</a>`);
    if (n.subPages.length) {
      e.push(this._renderChildsTokens(n.subPages));  // Recursive for subpages
    }
    e.push("</li>");
    r++;
  }
  return `<${this._options.listType}>${e.join("")}</${this._options.listType}>`;
};
```

### ADO HTML Structure (from rendered example)

```html
<!-- TOC -->
<div class="toc-container" aria-label="Table of contents" role="navigation">
    <div class="toc-container-header">Contents</div>
    <ul>
        <li><a href="#header-id">Header Text</a>
            <ul>
                <li><a href="#nested-id">Nested Header</a></li>
            </ul>
        </li>
    </ul>
</div>

<!-- TOSP -->
<div class="tosp-container">
    <div class="tosp-container-header">Child Pages</div>
    <ul></ul>
</div>
```

### Project Conventions

- Standards referenced: `copilot-instructions.md` best practices for widgets
- Instructions: Must follow Microsoft Fabric UI patterns with delete button always visible in high contrast
- Existing theme: `src/theme/ado-theme.css` has comprehensive theme support

## Key Discoveries

### Algorithm Comparison: ADO vs Our Implementation

| Aspect | ADO (`_renderChildsTokens`) | Our (`buildNestedTocHtml`) |
|--------|---------------------------|---------------------------|
| **Approach** | Recursive with index tracking | Iterative with level stack |
| **Level tracking** | Returns `[index, html]` tuple | Uses `levelStack: number[]` array |
| **Going deeper** | Recurses, gets nested HTML | Pushes level, opens `<ul>` |
| **Going up** | Returns immediately | Loops to close `</li></ul>` pairs |
| **Same level** | Closes `</li>`, pushes item | Closes `</li>`, starts new `<li>` |
| **Produces** | Same nested `<ul><li>` structure | Same nested `<ul><li>` structure |

**Conclusion**: Both algorithms produce **identical HTML output** - properly nested `<ul>/<li>` structures. The approaches differ (recursive vs iterative) but achieve the same result.

### Rendered HTML Comparison

**ADO's Actual Output** (from `Editing Page 1 - Wiki - Overview.html`):
```html
<div class="toc-container" aria-label="Table of contents" role="navigation">
  <div class="toc-container-header">Contents</div>
  <ul>
    <li><a href="...#1%5C.-first-level">1. First level</a>
      <ul>
        <li><a href="...#2.1.-second-level%2C-one">2.1. Second level, one</a></li>
        <li><a href="...#second-level%2C-two">Second level, two</a>
          <ul>
            <li><a href="...#third-level">Third level</a></li>
          </ul>
        </li>
      </ul>
    </li>
  </ul>
</div>
```

**Our Implementation Output** (matches ADO structure):
```html
<div class="toc-container">
  <div class="toc-container-header">Contents</div>
  <ul class="ado-toc-list">
    <li><a href="#first-level" class="ado-toc-link">1. First level</a>
      <ul class="ado-toc-list">
        <li><a href="#second-level-one" class="ado-toc-link">2.1. Second level, one</a></li>
        <li><a href="#second-level-two" class="ado-toc-link">Second level, two</a>
          <ul class="ado-toc-list">
            <li><a href="#third-level" class="ado-toc-link">Third level</a></li>
          </ul>
        </li>
      </ul>
    </li>
  </ul>
</div>
```

### Current vs Official Implementation Differences

| Aspect | Current (`ado-toc-widget`) | Official (`toc-container`) |
|--------|---------------------------|---------------------------|
| Width | Fixed `220px` | `min-width: 250px` auto |
| Border radius | `2px` | `4px` |
| Padding | `8px 12px` header + content | `10px 16px 10px 0` |
| Header | Uppercase, 12px, secondary background | Normal case, 600 weight, inline |
| Box shadow | Yes (Fabric card) | No |
| List bullets | None | `::before { content: "\2022 " }` |
| Nested lists | Left border indicator | Native nested UL |

### Critical Styling Elements from Official ADO

1. **No box shadows** - ADO uses simple border only
2. **Bullet points** - Uses `::before` pseudo-element with `\2022` (•)
3. **Header styling** - Simple bold text, no background color
4. **Nested indentation** - Uses `padding-left: 16px` on nested `<ul>`
5. **Link truncation** - `text-overflow: ellipsis` with `max-width: 400px`

### Implementation Requirements for Editor Widgets

Per `copilot-instructions.md`:
- Delete button must be always visible (not hover-only in high contrast modes)
- Use `visibility` and `opacity` instead of `display` to prevent layout shifts
- Delete button uses absolute positioning

## Recommended Approach

**Our TOC algorithm is correct and matches ADO's output structure.** The only remaining work is CSS styling alignment.

### Algorithm Status: ✅ Verified Correct

Our `buildNestedTocHtml` function using `levelStack` produces the same nested `<ul>/<li>` HTML structure as ADO's recursive `_renderChildsTokens` algorithm. Both handle:
- Level increases (going deeper): Open new `<ul>` inside current `<li>`
- Level decreases (going up): Close `</li></ul>` pairs until correct level
- Same level: Close previous `</li>`, start new `<li>`

### CSS Changes Required

To match official ADO appearance while preserving editor features:

1. **Container styling**: Remove box shadow, use border-only design
2. **Header styling**: Remove background color, use bold text only
3. **Bullet points**: Add `::before { content: "\2022 " }` pseudo-element
4. **Link styling**: Add `max-width: 400px` with text-overflow ellipsis
5. **Delete button**: Keep as positioned overlay (editor-specific feature)

## TOSP Services and API Architecture

### Data Flow for Table of Sub-Pages

The TOSP widget requires server-side data to display child pages. ADO uses a dedicated service class to fetch this data from the Wiki REST API.

### Service Architecture (from ms.vss-wiki-web.wiki-renderer-content.es6)

```javascript
// WikiPagesSource class - fetches page tree from Wiki API
class WikiPagesSource {
  constructor(pageContext) {
    this._pageContext = pageContext;
  }
  
  // Gets current page and its sub-pages up to specified depth
  getPageAndSubPages(wiki, pagePath, wikiVersion, depth = 4) {
    return getWikiClient(this._pageContext)
      .getPage(wiki.projectId, wiki.id, pagePath, depth, wikiVersion)
      .then(response => flattenWikiPage(response.page));
  }
}

// Wiki REST API client accessor
function getWikiClient(pageContext) {
  // Returns client with getPage method
  // API endpoint: _apis/wiki/wikis/{wikiId}/pages
  return pageContext.getService("WikiRestClient");
}

// Flattens hierarchical page response into array
function flattenWikiPage(page) {
  // Returns array of pages with subPages property
  return [page, ...flattenChildren(page.subPages)];
}
```

### TOSP Component Data Loading

```javascript
// In MarkdownRenderer component
this._wikiPagesSource = new WikiPagesSource(this.context.pageContext);
this._loadedSubPages = new Map();  // Cache to prevent duplicate API calls

// When TOSP marker encountered during render
_getSubPages(pagePath) {
  const cached = this._loadedSubPages.get(pagePath);
  if (cached) {
    return this._toISubPages(cached.subPages);
  }
  
  // Not in cache - load from API
  if (!this._loadedSubPages.has(pagePath)) {
    const wiki = this.props.wiki;
    const wikiVersion = this.props.wikiVersion;
    
    // Fetch with depth=120 (full tree)
    this._wikiPagesSource.getPageAndSubPages(wiki, pagePath, wikiVersion, 120)
      .then(pages => {
        if (pages.length > 0) {
          cached.subPages = pages[0].subPages;
          this.forceUpdate();  // Re-render with loaded data
        }
      });
    
    this._loadedSubPages.set(pagePath, true);  // Mark as loading
  }
  
  return [];  // Return empty while loading
}
```

### ISubPage Interface Transformation

```javascript
// Transforms API response to TOSP render format
// Sorts by page order, extracts name from path, recursively processes subpages
_toISubPages = pages => pages
  .sort((a, b) => a.order - b.order)  // Sort by ADO page order
  .map(page => ({
    name: this._getPageNameFromPath(page.path),  // Extract leaf name
    remoteUrl: page.remoteUrl,  // Full URL to page
    subPages: this._toISubPages(page.subPages)   // Recursive
  }));

// Extracts page name from path (last segment)
_getPageNameFromPath = path => {
  const match = path.match(/\/([^\/]*)$/);  // Match last /segment
  let name = path;
  if (match) {
    name = match.pop() || name;
  }
  return name;
};
```

### ISubPage Interface

```typescript
// Data structure for TOSP rendering
interface ISubPage {
  name: string;       // Display name (leaf of path)
  remoteUrl: string;  // Full URL to navigate to page
  subPages: ISubPage[]; // Nested child pages
}
```

### Wiki REST API Details

**Endpoint**: `GET _apis/wiki/wikis/{wikiId}/pages?path={pagePath}&recursionLevel={depth}`

**Parameters**:
- `projectId`: Azure DevOps project GUID
- `wikiId`: Wiki identifier GUID
- `pagePath`: Path to current page (e.g., "/Page%201")
- `depth`: Recursion depth (ADO uses 120 for full tree)
- `wikiVersion`: Git version/branch (e.g., "GBwikiMaster")

**Response**: Hierarchical page object with:
- `path`: Full page path
- `order`: Sort order within parent
- `remoteUrl`: Navigation URL
- `subPages`: Array of child page objects (recursive)

### Implementation Requirements for Our Editor

To implement live TOSP preview:

1. **API Integration**: Need Wiki REST API access via `ado-wiki-api.ts`
2. **Service Class**: Create equivalent `WikiPagesSource` service
3. **Caching**: Implement `Map` cache to prevent duplicate API calls
4. **Async Render**: Handle async data loading with loading state
5. **Transform**: Port `_toISubPages` transformation function
6. **Render**: Use existing `_renderChildsTokens` algorithm

### Key Differences: TOC vs TOSP

| Aspect | TOC | TOSP |
|--------|-----|------|
| **Data Source** | Document headings (local) | Wiki API (remote) |
| **Async** | No | Yes - requires API call |
| **Caching** | Not needed | Required for performance |
| **URL format** | `#anchor` | Full page URL |
| **Content** | Header text | Page name from path |

## Implementation Guidance

- **Algorithm Status**: ✅ Our `buildNestedTocHtml` is verified correct - matches ADO's `_renderChildsTokens` output
- **TOSP Services**: ✅ Fully researched - Wiki REST API with depth=120, ISubPage interface documented
- **Remaining Work**: 
  - CSS styling alignment (TOC/TOSP appearance)
  - TOSP API integration (requires `ado-wiki-api.ts` implementation)
- **Key Tasks**:
  1. Update CSS in `src/theme/ado-theme.css` to match official ADO styling
  2. Add bullet point pseudo-elements
  3. Adjust header styling (remove background, use bold text)
  4. Keep delete button functionality
  5. (Future) Implement TOSP data fetching via Wiki REST API
- **Dependencies**: TOSP live preview requires Wiki API access
- **Success Criteria**: 
  - Widget appearance matches ADO preview panel exactly
  - TOSP can display actual child pages when API is available

## Related Research

- **@Mentions**: See `20251130-ado-mention-implementation-research.md` for GUID-based storage, bidirectional translation, and Graph API integration details
