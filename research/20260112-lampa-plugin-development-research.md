<!-- markdownlint-disable-file -->

# Task Research Notes: Lampa Plugin Development

## Research Executed

### Documentation Analysis

- **API Documentation** (`docs/api.md`)
  - Core API reference covering globals (Lampa namespace), Storage, Reguest, Controller, Activity, Player
  - Network handling patterns with timeouts and headers
  - Settings and SettingsApi integration methods
  - Component lifecycle: create, start, back, render, destroy
  - Event and listener patterns for app lifecycle hooks

- **Architecture Documentation** (`docs/architecture.md`)
  - Plugin model and loading mechanisms
  - Lifecycle phases: load time, activation, deactivation
  - Hook and event system overview
  - Component architecture patterns
  - File structure recommendations
  - Versioning and compatibility guidelines

- **UI Documentation** (`docs/ui.md`)
  - Menu entry integration patterns (Settings and sidebar)
  - Scrollable list/grid implementation using Scroll component
  - Card and grid best practices
  - Details/info screen patterns
  - Background management
  - Player integration for UI components

- **Streams Documentation** (`docs/streams.md`)
  - Supported source formats (HLS, DASH, progressive MP4, TorrServer)
  - Player API usage patterns
  - Failover strategy implementation
  - Subtitles and audio track handling
  - CORS and proxy considerations
  - DRM limitations and fallbacks
  - Error handling checklist

- **Storage and Preferences Documentation** (`docs/storage-and-prefs.md`)
  - Storage conventions and namespacing
  - Schema definition with defaults
  - Version migration patterns
  - SettingsApi field types and integration
  - Storage change listeners

- **Security Documentation** (`docs/security.md`)
  - Risky patterns to avoid (eval, open proxies, auto-start features)
  - Safer alternatives and best practices
  - Privacy and legal compliance considerations

- **Testing and Debugging Documentation** (`docs/testing-debugging.md`)
  - Logging best practices with debug toggles
  - Focus and Controller debugging techniques
  - Network troubleshooting approaches
  - Player testing strategies
  - Performance optimization tips
  - Common pitfalls

- **Packaging and Publishing Documentation** (`docs/packaging-publishing.md`)
  - File structure (manifest.json, main.js, ui.js, styles.css)
  - Semantic versioning requirements
  - Distribution methods (HTTPS hosting, landing pages, checksums)

- **Migration Documentation** (`docs/migration.md`)
  - API deprecation tracking
  - Feature detection and fallback patterns
  - Changelog maintenance requirements

- **Plugin Catalog** (`docs/catalog.md`)
  - Comprehensive plugin catalog with security analysis
  - 40+ plugin summaries with integration patterns and risk assessments

### Security Audit Analysis

- **Security Audit Report** (`audit_report.md`)
  - Security audit findings for sampled plugins
  - High-risk patterns: eval() usage, obfuscated loaders, remote code execution
  - Medium-risk patterns: HTTP usage, token storage, proxy dependencies
  - Low-risk patterns: UI-only modifications, configuration toggles
  - Security recommendations for plugin developers

### Templates and Examples

- **Plugin Template** (`templates/plugin-template/`)
  - Production-ready starter template with manifest.json, main.js, ui.js, styles.css
  - Complete component lifecycle implementation
  - Settings integration example
  - Network request patterns

- **Sample Plugins** (`templates/sample-plugins/`)
  - Six specialized plugin examples: IPTV, search provider, settings-only, TMDB proxy, UI tweak, details-episodes
  - Practical implementation patterns for common use cases

- **Full Usage Examples** (`examples/full-usage-examples.md`)
  - End-to-end code snippets for common tasks
  - Settings entry and component creation
  - Network requests with headers
  - Search hook implementation

- **Plugin Checklist** (`checklists/plugin-checklist.md`)
  - Pre-publication checklist covering metadata, security, storage, network, player, and licensing

## Key Discoveries

### Plugin Architecture

#### Core Plugin Model

Lampa plugins are JavaScript files loaded into the app's WebView context with access to global namespaces:

- **Entry Point**: Plain JavaScript file loaded via direct URL or local install
- **Execution Context**: Browser JavaScript in WebView with access to globals
- **Registration**: Self-registration with Lampa application context via `Lampa.Component.add()` or `Lampa.Plugin.create()`

**Global Namespaces Available**:
- `Lampa` - Main namespace with submodules
- `Utils`, `Storage`, `Component`, `Template`, `Network`, `Controller`, `Player`
- `Settings`, `Activity`, `Favorites`, `Files`, `Platform`

#### Plugin Lifecycle

```
Load Time → Activation → Runtime → Deactivation
    ↓           ↓          ↓            ↓
Execute     Register   Handle       Cleanup
  Once      Hooks      Events      Resources
```

**Load Time**: Executed once when added; may hook events for app start, search, route changes

**Activation**: Usually by calling `Lampa.Plugin.create()` or registering screens/menus via `Lampa.Listener`

**Runtime**: Respond to events, handle user interactions, manage component lifecycle

**Deactivation**: Remove listeners, controllers, intervals; respect `onDestroy` patterns

#### Component Lifecycle Methods

Components must implement these methods:

```javascript
function MyComponent(object){
  // Required methods
  this.create = () => {
    // Build UI, load data, initialize
    // Return this.render()
  }
  
  this.start = () => {
    // Register controller, set focus
    // Called after create
  }
  
  this.render = () => {
    // Return HTML element
    return html
  }
  
  this.destroy = () => {
    // Cleanup: remove listeners, clear network, destroy elements
  }
  
  // Optional methods
  this.back = () => {
    // Handle back button
    Lampa.Activity.backward()
  }
  
  this.pause = () => {
    // Handle pause event
  }
  
  this.resume = () => {
    // Handle resume event
  }
}
```

#### File Structure Recommendations

```
plugin-name/
├── manifest.json     # Plugin metadata (id, name, version, entry)
├── main.js          # Registration, routes, settings
├── ui.js            # UI components or templates (optional)
└── styles.css       # Optional styling tweaks
```

**manifest.json** structure:

```json
{
  "id": "myplugin",
  "name": "My Lampa Plugin",
  "version": "0.1.0",
  "entry": "main.js",
  "author": "Your Name",
  "description": "Plugin description",
  "homepage": "https://example.com",
  "permissions": ["network", "storage"]
}
```

### Core API Reference

#### Storage API

Simple key/value persistence with change listeners.

**Best Practices**:
- Namespace keys with plugin ID: `myplugin_*`
- Define schema with defaults and merge on read
- Version your preferences for migration support

```javascript
const KEY = 'myplugin_prefs'
const defaults = { enabled: true, quality: 'auto', theme: 'dark', v: 1 }

// Load with defaults and migration
function load(){
  const saved = Lampa.Storage.get(KEY, {})
  const prefs = Object.assign({}, defaults, saved)
  if ((saved.v||0) < defaults.v) migrate(prefs, saved.v||0)
  return prefs
}

// Save preferences
function save(prefs){ 
  Lampa.Storage.set(KEY, prefs) 
}

// Migration handler
function migrate(prefs, from){
  if (from < 1) {
    prefs.theme = prefs.theme || 'dark'
    prefs.v = 1
  }
}

// Listen for changes
Lampa.Storage.listener.follow('change', (e) => {
  if (e.name === KEY) {
    // React to live changes, e.value holds new value
  }
})

// Read built-in fields (not persisted via set)
const isPremium = Lampa.Storage.field('account_premium')
```

#### Network API (Reguest)

HTTP client with timeout, headers, and convenience methods.

```javascript
const network = new Lampa.Reguest()
network.timeout(10000)

// GET request (silent: no global spinners)
network.silent('https://api.example.com/items', (json) => {
  // Success callback
}, (err) => {
  // Error callback
  Lampa.Noty.show('Network error')
})

// POST with headers
network.silent('https://api.example.com/items', 
  (json)=>{ /* success */ }, 
  (err)=>{ /* error */ }, 
  false, 
  {
    type: 'POST',
    headers: { 
      'x-api-key': '...',
      'Authorization': 'Bearer ...'
    },
    data: JSON.stringify({ q: 'query' })
  }
)

// Cleanup in component.destroy
network.clear()
```

**Network Best Practices**:
- Set timeouts (5-10 seconds)
- Use HTTPS endpoints
- Clear network on component destroy
- Surface errors with `Lampa.Noty.show()`

#### Settings and SettingsApi

Two levels of settings integration:

**1. Settings Entry** (opens plugin screen):

```javascript
Lampa.Settings.add({
  title: 'My Plugin',
  group: 'plugins',
  subtitle: 'Enable or configure features',
  onSelect: () => Lampa.Activity.push({ 
    title: 'My Plugin', 
    component: 'myplugin' 
  })
})
```

**2. Structured Settings Fields**:

```javascript
// Toggle field
Lampa.SettingsApi.addParam({
  component: 'myplugin',
  param: { 
    type: 'trigger', 
    name: 'myplugin_enabled', 
    default: true 
  },
  field: { name: 'Enable plugin' }
})

// Select field
Lampa.SettingsApi.addParam({
  component: 'myplugin',
  param: { 
    type: 'select', 
    name: 'myplugin_quality', 
    values: ['auto','720p','1080p'], 
    default: 'auto' 
  },
  field: { name: 'Preferred quality' }
})
```

#### Activity and Navigation

Navigation stack with push/replace/backward and lifecycle hooks.

```javascript
// Push new screen
Lampa.Activity.push({ 
  title: 'My Plugin', 
  component: 'myplugin',
  page: 1,
  url: 'custom-route'
})

// Go back
Lampa.Activity.backward()

// Replace current screen
Lampa.Activity.replace({ 
  title: 'New Screen', 
  component: 'other' 
})
```

**Activity Lifecycle Hooks**:
- `this.activity.loader(true/false)` - Show/hide loading spinner
- `this.activity.toggle()` - Activate the activity

#### Controller and Focus Management

Handle focus and key events for TV remotes and keyboard navigation.

```javascript
this.start = () => {
  Lampa.Controller.add('myplugin', {
    toggle(){
      Lampa.Controller.collectionSet(scroll.render())
      Lampa.Controller.collectionFocus(items[0]?.[0] || false, scroll.render())
    },
    back: this.back,
    up(){ Navigator.move('up') },
    down(){ Navigator.move('down') },
    left(){ 
      if (Navigator.canmove('left')) Navigator.move('left')
      else Lampa.Controller.toggle('menu') 
    },
    right(){ Navigator.move('right') }
  })
  Lampa.Controller.toggle('myplugin')
}
```

**Focus Requirements**:
- Elements must have `.selector` class to be focusable
- Call `scroll.update(el)` on `hover:focus` event
- Use `collectionFocus` with safe fallback

#### Player API

Open player with URL, playlists, callbacks, subtitles, and quality options.

```javascript
// Simple playback
Lampa.Player.play({ 
  title: 'Sample', 
  url: 'https://cdn.example/video.m3u8' 
})

// Playlist with multiple qualities
const sources = [
  { quality: '720p', url: 'https://cdn.example/720.m3u8' },
  { quality: '1080p', url: 'https://cdn.example/1080.m3u8' }
]

Lampa.Player.play({ title: 'Sample', url: sources[1].url })
Lampa.Player.playlist(sources.map(s => ({ 
  title: `Sample ${s.quality}`, 
  url: s.url 
})))

// Player callbacks
Lampa.Player.callback((e) => {
  if (e.type === 'destroy') {
    // Cleanup after playback ends
  }
  if (e.type === 'error') {
    // Handle playback error
  }
})
```

#### Event System

React to app lifecycle and custom events.

```javascript
// App lifecycle events
Lampa.Listener.follow('app', (e) => {
  if (e.type === 'ready') {
    // Initialize plugin when app is ready
  }
  if (e.type === 'background') {
    // Handle app going to background
  }
})

// Settings events
Lampa.Settings.listener.follow('open', (e) => {
  if (e.name === 'myplugin') {
    // Mutate settings screen if needed
  }
})

// Storage change events
Lampa.Storage.listener.follow('change', (e) => {
  if (e.name === 'myplugin_prefs') {
    // React to preference changes
  }
})

// Search events
Lampa.Listener.follow('search', (e) => {
  if (e.query) {
    // Provide custom search results
  }
})
```

### UI Integration Patterns

#### Adding Menu Entries

**Settings Entry**:

```javascript
Lampa.Settings.add({
  title: 'My Plugin',
  group: 'plugins',
  onSelect: () => Lampa.Activity.push({ 
    title: 'My Plugin', 
    component: 'myplugin' 
  })
})
```

**Sidebar Button**:

```javascript
function addSidebar(){
  const button = $('<li class="menu__item selector">\
    <div class="menu__ico">📦</div>\
    <div class="menu__text">My Plugin</div>\
  </li>')
  
  button.on('hover:enter', () => {
    Lampa.Activity.push({ 
      title: 'My Plugin', 
      component: 'myplugin' 
    })
  })
  
  $('.menu .menu__list').eq(0).append(button)
}

if (window.appready) addSidebar()
else Lampa.Listener.follow('app', e => { 
  if (e.type === 'ready') addSidebar() 
})
```

#### Scrollable Lists and Grids

Use `Scroll` component with `.selector` elements for focusable items.

```javascript
function ListComponent(){
  const scroll = new Lampa.Scroll({ mask: true, over: true })
  const html = $('<div class="myplugin">\
    <div class="myplugin__body"></div>\
  </div>')
  const body = html.find('.myplugin__body')
  let items = []

  this.create = () => {
    body.append(scroll.render(true))
    
    // Add items
    ;['One','Two','Three'].forEach((title) => {
      const el = $('<div class="selector myplugin__item"></div>')
        .text(title)
      
      el.on('hover:enter', () => {
        Lampa.Noty.show('Selected '+title)
      })
      
      el.on('hover:focus', () => {
        scroll.update(el)
      })
      
      scroll.append(el)
      items.push(el)
    })
    
    this.activity.toggle()
    return this.render()
  }

  this.start = () => {
    Lampa.Controller.add('myplugin', {
      toggle(){ 
        Lampa.Controller.collectionSet(scroll.render())
        Lampa.Controller.collectionFocus(items[0]?.[0] || false, scroll.render())
      },
      back: this.back,
      up(){ Navigator.move('up') },
      down(){ Navigator.move('down') },
      left(){ 
        if (Navigator.canmove('left')) Navigator.move('left')
        else Lampa.Controller.toggle('menu') 
      },
      right(){ Navigator.move('right') }
    })
    Lampa.Controller.toggle('myplugin')
  }

  this.back = () => Lampa.Activity.backward()
  this.render = () => html
  this.destroy = () => {
    Lampa.Arrays.destroy(items)
    scroll.destroy()
    html.remove()
  }
}
```

**Grid Best Practices**:
- Use existing card templates for consistent look
- Ensure elements have `.selector` class for focus
- Keep 10-15 items per row for performance
- Paginate large datasets
- Destroy resources in `destroy()`

#### Details/Info Screens

Pattern for detail screens with backdrop, metadata, and actions:

```javascript
const info = $('<div class="myplugin-info">\
  <div class="myplugin-info__title">Title</div>\
  <div class="myplugin-info__meta">2024 • Action</div>\
  <div class="myplugin-info__actions">\
    <div class="selector myplugin-btn">Play</div>\
    <div class="selector myplugin-btn">Add to Favorites</div>\
  </div>\
</div>')
```

#### Background Management

Switch backgrounds for immersive experience:

```javascript
Lampa.Background.change('https://image.tmdb.org/t/p/w1280/abc.jpg')
```

#### Templates

Use predefined templates or create custom DOM:

```javascript
// Using built-in templates
const card = Lampa.Template.get('card', { 
  title: 'Item',
  poster: 'https://...'
})

// Custom DOM
const row = $('<div class="myplugin-row"></div>')
```

### Streams and Playback

#### Supported Source Formats

- **HLS (.m3u8)**: Best cross-device compatibility, adaptive bitrate
- **DASH (.mpd)**: May not work on all WebViews, test before using
- **Progressive (.mp4)**: Simple but no adaptive bitrate
- **TorrServer**: Local streaming via torrent engine (legal/privacy risk; user opt-in only)

#### Failover Strategy

Implement multiple mirrors with auto-retry on errors:

```javascript
const mirrors = [
  'https://a.example/1080.m3u8',
  'https://b.example/1080.m3u8',
  'https://c.example/1080.m3u8'
]
let idx = 0

function start(){ 
  Lampa.Player.play({ 
    title: 'Demo', 
    url: mirrors[idx] 
  }) 
}

Lampa.Player.callback((e) => {
  if (e.type === 'error' && idx < mirrors.length - 1) {
    idx++
    Lampa.Noty.show('Trying alternate source...')
    start()
  }
})

start()
```

#### Subtitles and Audio Tracks

```javascript
const subtitles = [
  { label: 'EN', url: 'https://cdn.example/en.vtt' },
  { label: 'ES', url: 'https://cdn.example/es.vtt' }
]

const audioTracks = [
  { label: 'ENG', url: 'https://cdn.example/audio_eng.m3u8' },
  { label: 'SPA', url: 'https://cdn.example/audio_spa.m3u8' }
]

// Implement selector UI and re-open player with chosen tracks
```

#### CORS and Proxy Considerations

- **Prefer HTTPS** with proper CORS from origin
- **If proxy required**: Disclose endpoint and purpose in Settings, allow opt-out
- **Avoid generic open proxies**: Pin hosts and sanitize query parameters
- **Proxy disclosure**: Users should understand what data passes through proxies

#### DRM Limitations

- Many WebViews lack Widevine/PlayReady support
- Detect capability and warn users if DRM streams are unavailable
- Provide graceful fallback (trailer, alternate source)

#### Error Handling Checklist

- Validate MIME type and file extension
- Implement retries with exponential backoff
- Provide mirror fallback options
- Time out slow requests (5-10 seconds)
- Surface user-friendly error messages via `Lampa.Noty.show()`
- Test multiple formats (m3u8, mp4, mpd)

### Security Best Practices

#### Risky Patterns to Avoid

**High Risk**:
- `eval()` / `new Function()` / `Function()` constructor
- Dynamic remote code execution
- Open proxies with CORS any-origin
- Auto-starting TorrServer without user consent
- Disabling WebSocket protections or tampering with CSP
- Hardcoded tokens, API keys, or user identifiers

**Medium Risk**:
- HTTP endpoints (vulnerable to MITM)
- Storing long-lived tokens in plain Storage
- Obfuscated code that hides behavior
- Extensive proxying through third-party services

#### Safer Approaches

```javascript
// ✅ GOOD: Bundle static code
(function(){
  const ID = 'myplugin'
  // All code is visible and static
  function init(){ /* ... */ }
  init()
})()

// ❌ BAD: Remote code execution
fetch('https://remote.com/code.js')
  .then(r => r.text())
  .then(code => eval(code))

// ✅ GOOD: Validate and sanitize responses
network.silent(url, (json) => {
  if (!json || typeof json !== 'object') {
    return Lampa.Noty.show('Invalid response')
  }
  // Process validated data
}, (err) => {
  Lampa.Noty.show('Network error')
})

// ✅ GOOD: Opt-in for risky features
Lampa.SettingsApi.addParam({
  component: 'myplugin',
  param: { 
    type: 'trigger', 
    name: 'myplugin_enable_torrents', 
    default: false 
  },
  field: { 
    name: 'Enable TorrServer (Privacy Risk)',
    description: 'Warning: This feature may expose your IP address'
  }
})
```

#### Privacy and Legal Compliance

- **Do not collect PII** without explicit consent
- **Anonymous telemetry only**: Make it opt-in with clear disclosure
- **Respect regional laws**: Do not circumvent DRM or geo-blocks
- **Disclose all external services**: What data is sent, where, and why
- **Token handling**: Don't store long-lived secrets unencrypted
- **HTTPS only**: Avoid HTTP endpoints that expose user data

#### Security Audit Findings

Based on analysis of 40+ community plugins:

**High-Risk Patterns Found**:
- Remote code execution via `eval()` (3 plugins)
- Obfuscated loaders from raw IPs over HTTP (2 plugins)
- Console method stubbing to hide behavior (2 plugins)

**Medium-Risk Patterns Found**:
- HTTP endpoints for API calls (10+ plugins)
- Plain-text token storage (5+ plugins)
- Extensive third-party proxying (3 plugins)
- WebSocket bridges for remote execution (2 plugins)

**Recommendations**:
- Disallow `eval()` and dynamic code execution unless signed and verified
- Require HTTPS for all external endpoints
- Encrypt sensitive data in Storage
- Limit or sandbox WebSocket bridges
- Document and disclose all third-party endpoints

### Testing and Debugging

#### Logging Best Practices

```javascript
const DEBUG = Lampa.Storage.get('myplugin_debug', false)

function log(msg){
  if (DEBUG) console.log('[myplugin]', msg)
}

// Add debug toggle in settings
Lampa.SettingsApi.addParam({
  component: 'myplugin',
  param: { 
    type: 'trigger', 
    name: 'myplugin_debug', 
    default: false 
  },
  field: { name: 'Enable debug logging' }
})
```

#### Focus and Controller Debugging

```javascript
// Ensure component calls Controller.add and toggle
this.start = () => {
  Lampa.Controller.add('myplugin', {
    toggle(){ 
      const collection = scroll.render()
      Lampa.Controller.collectionSet(collection)
      
      // Safe fallback if no items
      const firstItem = items[0]?.[0] || false
      Lampa.Controller.collectionFocus(firstItem, collection)
      
      console.log('[myplugin] Focus set to:', firstItem)
    },
    // ...
  })
  Lampa.Controller.toggle('myplugin')
}

// Verify .selector elements exist
console.log('Selectors found:', $('.selector').length)
```

#### Network Troubleshooting

```javascript
const network = new Lampa.Reguest()
network.timeout(8000)

network.silent('/mock/data.json', 
  (json) => {
    console.log('[myplugin] Response:', json)
  }, 
  (err) => {
    console.error('[myplugin] Network error:', err)
    Lampa.Noty.show('Network error')
  }
)

// Stub external calls during development
const MOCK_MODE = true
const url = MOCK_MODE ? '/mock/data.json' : 'https://api.example.com/items'
```

#### Player Troubleshooting

```javascript
// Test multiple formats and mirrors
const sources = [
  { quality: '720p', url: 'https://a.example/720.m3u8', format: 'hls' },
  { quality: '720p', url: 'https://b.example/720.mp4', format: 'mp4' }
]

Lampa.Player.callback((e) => {
  console.log('[myplugin] Player event:', e.type)
  
  if (e.type === 'error') {
    console.error('[myplugin] Playback error:', e)
    // Try alternate source or format
  }
  
  if (e.type === 'destroy') {
    console.log('[myplugin] Player destroyed, cleanup UI')
    // Restore UI state
  }
})
```

#### Performance Optimization

- **Avoid huge DOM trees**: Paginate and lazy-load
- **Reuse Scroll instances**: Don't create multiple scrolls unnecessarily
- **Destroy event handlers**: Always clean up in `destroy()`
- **Limit items per render**: 10-15 items per row, paginate the rest
- **Debounce expensive operations**: Use timeouts for search, filtering

```javascript
// Lazy loading example
let currentPage = 0
const itemsPerPage = 20

function loadMore(){
  const start = currentPage * itemsPerPage
  const end = start + itemsPerPage
  const page = allItems.slice(start, end)
  
  page.forEach(item => {
    const el = createItemElement(item)
    scroll.append(el)
    items.push(el)
  })
  
  currentPage++
}
```

#### Common Pitfalls

- **CORS blocked**: Only use approved proxies and HTTPS endpoints
- **Obfuscated code**: Avoid `eval`/dynamic scripts; prefer static code
- **Token handling**: Don't store long-lived secrets unencrypted
- **Missing destroy**: Always implement `destroy()` to prevent memory leaks
- **Focus issues**: Ensure `.selector` class on focusable elements
- **HTTP usage**: Prefer HTTPS to avoid MITM attacks

### Packaging and Publishing

#### Required Files

```
plugin-package/
├── manifest.json     # Required: metadata
├── main.js          # Required: entry point
├── ui.js            # Optional: UI helpers
└── styles.css       # Optional: custom styles
```

#### Versioning

- Use **semantic versioning**: MAJOR.MINOR.PATCH
- Document breaking changes in `migration.md` or changelog
- Increment versions consistently:
  - MAJOR: Breaking changes
  - MINOR: New features (backward compatible)
  - PATCH: Bug fixes

```json
{
  "version": "1.2.3"
}
```

#### Distribution

1. **Host on HTTPS** with correct MIME type (`application/javascript`)
2. **Provide landing page** with:
   - Plugin description and features
   - Installation URL
   - Screenshots
   - Changelog
3. **Sign releases** or publish checksums (SHA256)
4. **Update manifest** with accurate homepage and author info

**Installation URL Format**:
```
https://yourdomain.com/plugins/myplugin/main.js
```

#### Pre-Publish Checklist

- [ ] Metadata complete: id, name, version, description
- [ ] Settings page provided with toggles for risky features
- [ ] Storage keys namespaced; defaults present
- [ ] Network calls documented; avoid open proxies
- [ ] CORS handled legally; no DRM circumvention
- [ ] Player tested with multiple formats and fallbacks
- [ ] Focus/Controller works with remote
- [ ] No eval/new Function; no remote code exec
- [ ] No hardcoded tokens; use config pattern if needed
- [ ] License included; third-party attributions listed
- [ ] QA smoke tests pass; logs are gated
- [ ] Documentation updated (README, changelog)
- [ ] Version number incremented appropriately

### Migration and Compatibility

#### API Deprecation Tracking

- Check for deprecations in `Lampa` namespaces you use
- Monitor Lampa changelogs and community forums
- Test on latest Lampa builds regularly

#### Feature Detection and Fallback

```javascript
// Feature detection pattern
function init(){
  if (!window.Lampa) {
    return console.error('[myplugin] Lampa not available')
  }
  
  // Check for specific API
  if (typeof Lampa.SettingsApi === 'undefined') {
    console.warn('[myplugin] SettingsApi not available, using fallback')
    // Use alternative method
  }
  
  // Proceed with initialization
  register()
}
```

#### Versioning and Changelog

Maintain changelog in your repository:

```markdown
## Changelog

### v1.2.0 (2024-01-15)
- Added multi-quality support
- Fixed focus issue on grid view
- Requires Lampa 3.5.0+

### v1.1.0 (2023-12-01)
- Added settings page
- Improved error handling
```

Note required Lampa app versions for compatibility.

### Complete Plugin Examples

#### Minimal Plugin Template

```javascript
(function(){
  const ID = 'myplugin'
  const TITLE = 'My Lampa Plugin'

  function Screen(){
    const network = new Lampa.Reguest()
    const scroll = new Lampa.Scroll({ mask: true, over: true })
    const html = $('<div class="myplugin-screen">\
      <div class="myplugin-screen__body"></div>\
    </div>')
    const body = html.find('.myplugin-screen__body')
    let items = []

    this.create = () => {
      this.activity.loader(true)
      body.append(scroll.render(true))
      
      network.timeout(8000)
      network.silent('https://api.example.com/items', 
        (json) => {
          this.append(json.items)
          this.activity.loader(false)
          this.activity.toggle()
        }, 
        (err) => {
          const empty = new Lampa.Empty({ 
            descr: 'Failed to load data' 
          })
          html.append(empty.render(true))
          this.start = empty.start
          this.activity.loader(false)
          this.activity.toggle()
        }
      )
      
      return this.render()
    }

    this.append = (list) => {
      list.forEach((it) => {
        const el = $('<div class="selector myplugin-item"></div>')
          .text(it.title)
        
        el.on('hover:enter', () => {
          Lampa.Noty.show('Selected: '+it.title)
        })
        
        el.on('hover:focus', () => {
          scroll.update(el)
        })
        
        scroll.append(el)
        items.push(el)
      })
    }

    this.start = () => {
      Lampa.Controller.add(ID, {
        toggle(){ 
          Lampa.Controller.collectionSet(scroll.render())
          Lampa.Controller.collectionFocus(
            items[0]?.[0] || false, 
            scroll.render()
          )
        },
        back: this.back,
        up(){ Navigator.move('up') },
        down(){ Navigator.move('down') },
        left(){ 
          if (Navigator.canmove('left')) Navigator.move('left')
          else Lampa.Controller.toggle('menu') 
        },
        right(){ Navigator.move('right') }
      })
      Lampa.Controller.toggle(ID)
    }

    this.back = () => Lampa.Activity.backward()
    this.render = () => html
    this.destroy = () => { 
      network.clear()
      Lampa.Arrays.destroy(items)
      scroll.destroy()
      html.remove()
    }
  }

  function addSettings(){
    Lampa.Settings.add({
      title: TITLE,
      group: 'plugins',
      subtitle: 'Enable or configure features',
      onSelect: () => Lampa.Activity.push({ 
        title: TITLE, 
        component: ID 
      })
    })
  }

  function init(){
    if (!window.Lampa){
      return console.log('[myplugin] Lampa not ready')
    }
    Lampa.Component.add(ID, Screen)
    addSettings()
  }

  if (window.appready) init()
  else Lampa.Listener.follow('app', (e) => { 
    if (e.type === 'ready') init() 
  })
})()
```

#### IPTV Plugin Example

```javascript
(function(){
  const ID = 'sample_iptv'
  const CHANNELS = [
    { title: 'Channel 1', url: 'https://example.com/1.m3u8' },
    { title: 'Channel 2', url: 'https://example.com/2.m3u8' }
  ]

  function IPTV(){
    this.create = () => {
      const list = $('<div class="iptv-list"></div>')
      
      CHANNELS.forEach(ch => {
        const item = $('<div class="selector iptv-item"></div>')
          .text(ch.title)
        
        item.on('hover:enter', () => {
          Lampa.Player.play({ 
            title: ch.title, 
            url: ch.url 
          })
        })
        
        list.append(item)
      })
      
      this.html = $('<div class="iptv-screen"></div>').append(list)
      this.start()
    }
    
    this.start = () => {
      Lampa.Controller.add(ID, { 
        back: () => Lampa.Activity.backward(),
        toggle(){ 
          Lampa.Controller.collectionSet($('.iptv-screen'))
          Lampa.Controller.collectionFocus(
            $('.iptv-item').get(0), 
            $('.iptv-screen')
          )
        } 
      })
      Lampa.Controller.toggle(ID)
    }
    
    this.destroy = () => {}
  }

  Lampa.Plugin.create(ID, { 
    title: 'Sample IPTV', 
    onStart(){ 
      Lampa.Activity.push({ 
        title: 'Sample IPTV', 
        component: ID 
      })
    } 
  })
})()
```

#### Search Provider Example

```javascript
(function(){
  const ID = 'sample_search_provider'

  function SearchComponent(){
    const scroll = new Lampa.Scroll({ mask: true, over: true })
    const html = $('<div class="search-provider">\
      <div class="search-provider__body"></div>\
    </div>')
    const body = html.find('.search-provider__body')
    let items = []

    this.create = () => {
      body.append(scroll.render(true))
      this.activity.toggle()
      return this.render()
    }
    
    this.showResults = (list) => {
      scroll.clear()
      items = []
      
      list.forEach(v => {
        const el = $('<div class="selector search-item"></div>')
          .text(v.title)
        
        el.on('hover:enter', () => {
          Lampa.Player.play({ title: v.title, url: v.url })
        })
        
        el.on('hover:focus', () => {
          scroll.update(el)
        })
        
        scroll.append(el)
        items.push(el)
      })
    }
    
    this.start = () => {
      Lampa.Controller.add(ID, {
        toggle(){ 
          Lampa.Controller.collectionSet(scroll.render())
          Lampa.Controller.collectionFocus(
            items[0]?.[0] || false, 
            scroll.render()
          )
        },
        back: this.back,
        up(){ Navigator.move('up') },
        down(){ Navigator.move('down') },
        left(){ 
          if (Navigator.canmove('left')) Navigator.move('left')
          else Lampa.Controller.toggle('menu') 
        },
        right(){ Navigator.move('right') }
      })
      Lampa.Controller.toggle(ID)
    }
    
    this.back = () => Lampa.Activity.backward()
    this.render = () => html
  }

  Lampa.Component.add(ID, SearchComponent)
  
  Lampa.Listener.follow('search', (e) => {
    if (e && e.query) {
      const comp = new SearchComponent()
      Lampa.Activity.push({ 
        title: 'Search: '+e.query, 
        component: comp 
      })
      
      const demo = [
        { 
          title: e.query + ' 720p', 
          url: 'https://cdn.example/720.m3u8' 
        },
        { 
          title: e.query + ' 1080p', 
          url: 'https://cdn.example/1080.m3u8' 
        }
      ]
      
      setTimeout(() => comp.showResults(demo), 50)
    }
  })
})()
```

#### Settings-Only Plugin Example

```javascript
(function(){
  const KEY = 'sample_flag'
  
  function toggle(){
    const next = !Lampa.Storage.get(KEY, false)
    Lampa.Storage.set(KEY, next)
    Lampa.Noty.show('Sample flag: ' + (next ? 'ON' : 'OFF'))
  }
  
  Lampa.Settings.add({ 
    title: 'Sample: Toggle Flag', 
    group: 'plugins', 
    subtitle: 'No UI screen', 
    onSelect: toggle 
  })
})()
```

## Implementation Notes

### Quick Start Checklist

1. **Copy plugin template** from `templates/plugin-template/`
2. **Customize IDs** in manifest.json and main.js
3. **Implement component lifecycle**: create, start, render, destroy
4. **Add Settings entry** for discoverability
5. **Test locally** using hello-world.html harness
6. **Host on HTTPS** and load in Lampa via URL
7. **Follow security guidelines**: no eval, validate inputs, use HTTPS

### Development Workflow

1. **Prototype** using `examples/hello-world.html` in browser
2. **Stub API calls** with local mock JSON during development
3. **Enable debug logging** with settings toggle
4. **Test focus navigation** on actual TV remote or keyboard
5. **Test multiple screen sizes** and orientations
6. **Profile performance** with large datasets
7. **Validate security** using checklist from audit report
8. **Version and publish** with proper semantic versioning

### Compatibility Considerations

- Use **ES5/ES6 compatible syntax** (WebView environments vary)
- **Feature-detect** Lampa APIs before using
- **Fallback gracefully** when APIs are unavailable
- Test on **multiple Lampa versions** and platforms
- Document **minimum required Lampa version**

### Performance Tips

- **Paginate** large lists (20-50 items per page)
- **Lazy-load** images and content as user scrolls
- **Reuse DOM elements** instead of recreating
- **Debounce** expensive operations (search, filtering)
- **Destroy resources** properly to prevent memory leaks
- **Profile with browser DevTools** to identify bottlenecks

## References

### Source Documentation

This research consolidates information from the following documentation sources:
- **Plugin Template**: Production-ready starter template with complete lifecycle implementation
- **Sample Plugins**: Six specialized plugin examples covering common use cases
- **Security Audit Report**: Analysis of 40+ real-world plugins with risk assessments
- **Plugin Catalog**: Comprehensive catalog of analyzed plugins with integration patterns

### External Resources

- Lampa GitHub: https://github.com/yumata/lampa
- Community Forums: Various plugin developers and users

### Key Topics Covered

- **Architecture**: Plugin model and lifecycle
- **API Reference**: Core APIs and usage patterns
- **UI Integration**: Menu entries, lists, grids, player
- **Streams**: Playback, failover, CORS, DRM
- **Storage**: Preferences and persistence
- **Security**: Risk mitigation and best practices
- **Testing**: Debugging techniques
- **Packaging**: Distribution guidelines
- **Migration**: Compatibility and versioning

## Conclusion

Lampa provides a flexible plugin system for extending the media player with custom sources, UI components, and integrations. The architecture is based on:

1. **Component Lifecycle**: Structured create/start/render/destroy pattern
2. **Event-Driven**: Listeners for app lifecycle, search, player events
3. **Focus Management**: Controller system for TV remote navigation
4. **Modular APIs**: Storage, Network, Player, Settings, Activity

**Key Success Factors**:
- Follow security best practices (no eval, HTTPS only, validate inputs)
- Implement proper lifecycle management (always destroy resources)
- Test focus navigation thoroughly with remote controls
- Provide user-friendly error handling and feedback
- Document all external dependencies and data flows
- Version consistently and maintain changelogs

**Common Pitfalls to Avoid**:
- Remote code execution via eval/dynamic scripts
- HTTP endpoints exposing user data
- Missing destroy implementation causing memory leaks
- Hardcoded credentials or tokens
- Poor error handling without user feedback
- Obfuscated code hiding security issues

**Recommended Development Path**:
1. Start with plugin template
2. Prototype with hello-world harness
3. Implement core functionality with proper lifecycle
4. Add settings and preferences
5. Test thoroughly on target devices
6. Security audit using checklist
7. Document and publish with proper versioning

The plugin ecosystem includes 40+ analyzed plugins ranging from simple UI tweaks to complex multi-source aggregators, with security audit findings highlighting the importance of following best practices for code safety, user privacy, and legal compliance.
