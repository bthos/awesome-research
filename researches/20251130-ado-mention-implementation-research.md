<!-- markdownlint-disable-file -->

# Task Research Notes: ADO @Mentions Implementation

## Research Executed

### File Analysis

- `.copilot-tracking/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-mention.mention-common.es6.EILvhD.min.js.download`
  - Core mention translation service with bidirectional display name ↔ storage key mapping
  - Contains `DisplayNameStorageKeyTranslationService` class
  - GUID validation and identity resolution logic

- `.copilot-tracking/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-mention.markdown-with-autocomplete.es6.pc0O4s.min.js.download`
  - Autocomplete UI component integration
  - Markdown preview with mention transformation

- `.copilot-tracking/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-mention.mention-autocomplete-resolver.es6.WGohwc.min.js.download`
  - Autocomplete contribution resolver service
  - Provider registration by shortcut character

- `.copilot-tracking/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wiki-renderer-content.es6.cuMuSd.min.js.download`
  - Wiki renderer with `MentionSyntaxProcessor` integration
  - Click handler registration for people and work item mentions

- `src/syntax/ado-mention-mark.ts`
  - Current implementation: Parses `@<user name>` format
  - Uses remark plugin and ProseMirror mark schema

### Code Search Results

- `@<` pattern in mention-common.es6
  - Found translation functions using `/\@\<(.*?)\>/g` regex
- `isGuid` function
  - Validates GUID format for storage keys
- `PeoplePickerProvider`
  - Resolves storage keys to identity objects via Graph API
- `MentionSyntaxProcessor`
  - Pre/post processing hooks in wiki renderer

## Key Discoveries

### Dual-Format System

ADO uses two formats for @mentions:

1. **Display Format**: `@<User Name>` - Human-readable, shown in editor
2. **Storage Format**: `@<GUID>` - UUID stored in markdown file

Example:
- Display: `@<John Doe>`
- Storage: `@<12345678-1234-1234-1234-123456789012>`

### Regex Patterns

```javascript
// General mention pattern - matches any @<content>
const mentionPattern = /\@\<(.*?)\>/g;

// Storage key (GUID) pattern - matches @<GUID> specifically
const storageKeyPattern = /\@\<([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})\>/g;

// GUID validation
function isGuid(value) {
  return /^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$/.test(value);
}
```

### DisplayNameStorageKeyTranslationService

Core service managing bidirectional translation:

```javascript
class DisplayNameStorageKeyTranslationService extends VssService {
  constructor() {
    // Map: displayName -> storageKey (GUID)
    this.displayNameToStorageKeyMap = {};
    
    // Map: storageKey -> { originalDisplayName, mangledDisplayName }
    this.storageKeyToDisplayNameMap = {};
  }
  
  // Add mapping between display name and storage key
  addDisplayNameAndStorageKey(displayName, storageKey) {
    if (storageKey && displayName && isGuid(storageKey)) {
      if (!this.isDisplayNameAndStorageKeyAlreadyAdded(storageKey, displayName)) {
        // Handle duplicate names by mangling
        const mangledName = this.checkIfDisplayNameExistsInMapAndMangle(displayName);
        
        this.displayNameToStorageKeyMap[mangledName] = storageKey.toUpperCase();
        this.storageKeyToDisplayNameMap[storageKey.toUpperCase()] = {
          originalDisplayName: displayName,
          mangledDisplayName: mangledName
        };
      }
    }
    return displayName;
  }
  
  // Handle duplicate display names: "John Doe" -> "John Doe(1)" -> "John Doe(2)"
  checkIfDisplayNameExistsInMapAndMangle(displayName) {
    let result = displayName;
    if (this.displayNameToStorageKeyMap[displayName]) {
      let counter = 1;
      result = displayName + "(" + counter + ")";
      while (this.displayNameToStorageKeyMap[result]) {
        counter++;
        result = displayName + "(" + counter + ")";
      }
    }
    return result;
  }
  
  // Get identity display name based on environment
  getIdentityId(identity) {
    // For hosted (Azure DevOps Services)
    if (this.isHosted()) {
      return identity.displayName || "";
    }
    // For on-prem (Azure DevOps Server)
    if (identity.entityType === "group") {
      return `[${identity.scopeName}]\\${identity.samAccountName}`;
    }
    return `${identity.scopeName}\\${identity.samAccountName}`;
  }
}
```

### Translation Functions

**Save (Display Names → Storage Keys):**
```javascript
translateDisplayNamesToStorageKeys(text) {
  const codeBlocks = parseCodeBlocksFromText(text);
  const containers = parseContainersFromText(text);
  const pattern = /\@\<(.*?)\>/g;
  const ignoreMatch = getIgnoreMatch(codeBlocks.concat(containers));
  
  let match;
  const matches = [];
  while (match = pattern.exec(text)) {
    matches.push(match);
  }
  
  // Process in reverse order to preserve indices
  for (let i = matches.length - 1; i >= 0; i--) {
    const m = matches[i];
    if (!ignoreMatch || !ignoreMatch(m.index)) {
      const storageKey = this.getStorageKeyUsingDisplayName(m[1]);
      if (storageKey) {
        text = text.substring(0, m.index) + `@<${storageKey}>` + text.substring(m.index + m[0].length);
      }
    }
  }
  return text;
}
```

**Load (Storage Keys → Display Names):**
```javascript
async translateStorageKeysToDisplayNames(text, escapeHtml = false) {
  const codeBlocks = parseCodeBlocksFromText(text);
  const containers = parseContainersFromText(text);
  const pattern = /\@\<(.*?)\>/g;
  const ignoreMatch = getIgnoreMatch(codeBlocks.concat(containers));
  
  let result = Promise.resolve(text);
  let match;
  const matches = [];
  while (match = pattern.exec(text)) {
    matches.push(match);
  }
  
  // Process in reverse order
  for (let i = matches.length - 1; i >= 0; i--) {
    const m = matches[i];
    const storageKey = m[1];
    
    if ((!ignoreMatch || !ignoreMatch(m.index)) && isGuid(storageKey)) {
      const startIndex = m.index;
      const length = m[0].length;
      
      // Check cache first, then fetch from API if needed
      if (this.storageKeyToDisplayNameMap[storageKey]) {
        result = result.then(text => {
          const displayName = this._getDisplayName(
            this.storageKeyToDisplayNameMap[storageKey].mangledDisplayName, 
            escapeHtml
          );
          return text.substring(0, startIndex) + displayName + text.substring(startIndex + length);
        });
      } else {
        // Fetch from Graph API
        result = result.then(text => 
          this.getIdentityUsingStorageKeyAsync(storageKey).then(identity => {
            if (identity) {
              const displayName = this._getDisplayName(this.getIdentityId(identity), escapeHtml);
              return text.substring(0, startIndex) + displayName + text.substring(startIndex + length);
            }
            return text;
          })
        );
      }
    }
  }
  return result;
}

// Format display name with optional HTML escaping
_getDisplayName(name, escapeHtml = false) {
  return escapeHtml ? `@&lt;${name}&gt;` : `@<${name}>`;
}
```

### Code Block Exclusion

Mentions inside code blocks and containers are NOT processed:

```javascript
// Parse code blocks to exclude from mention processing
function parseCodeBlocksFromText(text) {
  const results = [];
  const regex = /(```[a-z]*[\s\S]*?```)|(<pre>[a-z]*[\s\S]*?<\/pre>)|(`[a-z]*[\s\S]*?`)/ig;
  const newlinePattern = /\n[\s]*\n/ig;
  
  let match;
  while (match = regex.exec(text)) {
    let include = true;
    // Single backtick: exclude if contains double newlines
    if (text[match.index] === '`' && text[match.index + 1] !== '`') {
      const content = text.substring(match.index, regex.lastIndex);
      include = !newlinePattern.exec(content);
    }
    if (include) {
      results.push({ start: match.index, end: regex.lastIndex });
    }
  }
  return results;
}

// Parse ::: containers to exclude
function parseContainersFromText(text) {
  const results = [];
  const regex = /(:::[\s]*\S+[\s\r\n]+[\s\S]*?:::)/ig;
  
  let match;
  while (match = regex.exec(text)) {
    results.push({ start: match.index, end: regex.lastIndex });
  }
  return results;
}

// Check if position should be ignored
function getIgnoreMatch(excludeRanges) {
  if (excludeRanges.length) {
    return (index) => excludeRanges.some(range => index >= range.start && index <= range.end);
  }
  return null;
}
```

### Identity Resolution APIs

**PeoplePickerProvider:**
```javascript
class PeoplePickerProvider {
  constructor(pageContext) {
    this._pageContext = pageContext;
  }
  
  // Get single entity by storage key (GUID)
  getEntityFromUniqueAttribute(storageKey) {
    // Calls Graph API to resolve identity
    // Returns: { displayName, entityType, scopeName, samAccountName, originId, localId }
  }
  
  // Batch resolve multiple storage keys
  getEntitiesFromUniqueAttribute(storageKeys) {
    // Batch API call for efficiency
    // Returns array of identities
  }
}
```

**GraphRestClient:**
```javascript
class GraphRestClient {
  // Get storage key from subject descriptor
  async getStorageKey(subjectDescriptor) {
    return this.beginRequest({
      apiVersion: "5.0-preview.1",
      routeTemplate: "_apis/Graph/StorageKeys/{subjectDescriptor}",
      routeValues: { subjectDescriptor }
    });
  }
  
  // Create/materialize user identity
  async createUser(createRequest, groupDescriptors) {
    return this.beginRequest({
      apiVersion: "5.0-preview.1",
      method: "POST",
      routeTemplate: "_apis/Graph/Users/{userDescriptor}",
      queryParams: { groupDescriptors: groupDescriptors?.join(",") },
      body: createRequest
    });
  }
  
  // Create group
  async createGroup(createRequest, scopeDescriptor, groupDescriptors) {
    return this.beginRequest({
      apiVersion: "5.0-preview.1",
      method: "POST",
      routeTemplate: "_apis/Graph/Groups/{groupDescriptor}",
      queryParams: { scopeDescriptor, groupDescriptors: groupDescriptors?.join(",") },
      body: createRequest
    });
  }
}
```

### Identity Object Structure

```typescript
interface IIdentity {
  displayName: string;           // "John Doe"
  entityType: "user" | "group";  // Type of identity
  scopeName?: string;            // Domain or org scope (on-prem)
  samAccountName?: string;       // SAM account name (on-prem)
  originId: string;              // AAD object ID
  localId?: string;              // Storage key (GUID)
}
```

### MentionSyntaxProcessor

Wiki renderer integration:

```javascript
class MentionSyntaxProcessor {
  constructor(pageContext) {
    this._legacyMentionSyntaxProcessorPromise = 
      pageContext.getService("ILegacyPlatformService")
        .requireModules([
          "Mention/Scripts/MentionSyntaxProcessor",
          "Mention/Scripts/TFS.Mention.WorkItems.Registration",
          "Mention/Scripts/TFS.Mention.People.Registration"
        ])
        .then(([processor]) => new processor.MentionSyntaxProcessor());
  }
  
  // Pre-process markdown before rendering
  preProcess(markdown) {
    return this._legacyMentionSyntaxProcessorPromise
      .then(processor => processor.preProcess(markdown));
  }
  
  // Post-process rendered HTML
  postProcess(html) {
    return this._legacyMentionSyntaxProcessorPromise
      .then(processor => processor.postProcess(html));
  }
}
```

### Click Handler Registration

```javascript
// In wiki renderer initialization
this._getMentionsModulesPromise().then(([peopleModule, workItemModule]) => {
  const context = {
    SourcePageArea: "Wiki",
    wikiType: getWikiTypeString(this.props.wiki?.type || 0),
    newWiki: true
  };
  
  // People mentions - opens identity card
  peopleModule.PeopleMentionsRenderingProvider
    .registerMentionClickHandler(this._markdownContainerRef.current, context);
  
  // Work item mentions - opens work item dialog
  workItemModule.WorkItemMentionsRenderingProvider
    .registerMentionClickHandler(this._markdownContainerRef.current, context, clickCallback);
});
```

### Rendered HTML Structure

```html
<!-- User mention (rendered) -->
<a href="..." class="mention-link mention-people-link mention-click-handled">@John Doe</a>

<!-- Work item mention (rendered) -->
<a href="..." class="mention-link mention-wi-link mention-widget-workitem-no-access mention-click-handled">#75756</a>
```

### CSS Classes

From `ms.vss-wiki-web.wiki-renderer-content.min.css`:
```css
.mention-link { /* Base mention styling */ }
.mention-people-link { /* User mention specific */ }
.mention-wi-link { /* Work item mention specific */ }
.mention-widget-workitem-no-access { /* No permission indicator */ }
.mention-click-handled { /* Click handler attached */ }
```

## Comparison: Our Implementation vs ADO

| Aspect | Our Implementation | ADO Implementation |
|--------|-------------------|-------------------|
| **Format** | `@<User Name>` only | `@<GUID>` (stored) ↔ `@<Display Name>` (shown) |
| **Storage** | Display name in markdown | GUID stored, translated on load/save |
| **Resolution** | None (static text) | Graph API for identity lookup |
| **Caching** | None | Dual maps: displayName↔storageKey |
| **Duplicates** | Not handled | Name mangling: "John Doe(1)", "John Doe(2)" |
| **On-prem** | Not supported | `[scope]\\samAccountName` format |
| **Code blocks** | Processed (may be incorrect) | Excluded from processing |

## Serialization Note

Per `copilot-instructions.md`:
> When serializing markdown, unescape characters to properly display mentions. Specifically, replace `\<` with `<`, and `\.` with `.` in `@<user name>` mentions.

This is needed because remark may escape special characters in output.

## Recommended Approach

### For Standalone/Offline Use (Current)

Our current implementation is sufficient:
- Parse `@<User Name>` format
- Render as styled mention widget
- Serialize back to `@<User Name>`

### For Full ADO Integration (Future)

Would require:

1. **Translation Service**: Port `DisplayNameStorageKeyTranslationService`
2. **Graph API Integration**: Implement `PeoplePickerProvider` calls
3. **Caching Layer**: Dual maps for performance
4. **Code Block Exclusion**: Skip mentions in code blocks/containers
5. **Name Mangling**: Handle duplicate display names
6. **On-prem Support**: Domain\user format handling

## Implementation Guidance

- **Current Status**: ✅ Basic `@<User Name>` parsing and rendering works
- **Serialization Fix**: ✅ Unescape `\<` to `<` per project conventions
- **Future Enhancement**: Graph API integration for GUID translation (requires API access)
- **Dependencies**: Full ADO compatibility requires Graph API endpoint access
- **Success Criteria**: 
  - Mentions display correctly in editor
  - Mentions serialize without escaped characters
  - (Future) GUID translation when API available
