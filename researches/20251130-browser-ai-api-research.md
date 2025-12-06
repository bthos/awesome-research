<!-- markdownlint-disable-file -->

# Task Research Notes: Browser AI APIs for Extension Development

## Research Executed

### Web Documentation Analysis

- Chrome AI Built-in APIs (developer.chrome.com)
  - Comprehensive documentation on Prompt API, Translator, Summarizer, Writer/Rewriter
  - API status matrix with origin trial vs stable availability
  - Hardware requirements and free usage confirmed

- Chrome Extensions AI (developer.chrome.com/docs/extensions/ai)
  - Prompt API available in Chrome 138+ stable for extensions
  - Full integration documentation for extension development

- WebNN API (webmachinelearning.github.io/webnn/)
  - W3C standard for neural network inference hardware acceleration
  - Browser support: Chrome/Edge via DirectML, GPU, NPU backends
  - Low-level API for custom ML models (not high-level LLM access)

- Microsoft Windows AI (learn.microsoft.com/windows/ai/)
  - Phi Silica - Windows native SLM API (WinRT, not web)
  - Windows AI APIs require Copilot+ PC hardware
  - No direct browser extension API equivalent to Chrome's Prompt API

- Microsoft Edge Copilot (learn.microsoft.com/copilot/edge)
  - Copilot Chat sidebar feature (cloud-based, not local API)
  - No exposed JavaScript API for extension developers
  - Enterprise-focused with DLP integration

## Key Discoveries

### Chrome Built-in AI APIs (Gemini Nano)

#### API Status (as of Chrome 138)

| API | Web | Extensions | Status |
|-----|-----|------------|--------|
| **Prompt API** | Origin Trial | ✅ Chrome 138 Stable | Free, local Gemini Nano |
| **Translator API** | ✅ Chrome 138 Stable | ✅ Chrome 138 Stable | Free, local |
| **Language Detector API** | ✅ Chrome 138 Stable | ✅ Chrome 138 Stable | Free, local |
| **Summarizer API** | ✅ Chrome 138 Stable | ✅ Chrome 138 Stable | Free, local |
| **Writer API** | Origin Trial | Origin Trial | Free, local |
| **Rewriter API** | Origin Trial | Origin Trial | Free, local |
| **Proofreader API** | Origin Trial | Origin Trial | Free, local |

#### Cost Analysis: **COMPLETELY FREE**

- No API keys required
- No usage fees or quotas
- Model runs 100% locally on user's device
- No data sent to cloud servers
- Privacy-preserving by design

#### Hardware Requirements

| Component | Requirement |
|-----------|-------------|
| **Storage** | 22 GB free space (model downloads ~1.5GB compressed) |
| **GPU Option** | >4GB VRAM (dedicated or integrated) |
| **CPU Option** | 16GB RAM + 4 CPU cores (if no GPU) |
| **OS** | Windows 10/11, macOS 13+, Linux, ChromeOS |

#### Chrome Extension Usage Pattern

```javascript
// manifest.json - No special permissions needed for AI APIs
{
  "name": "My AI Extension",
  "version": "1.0",
  "manifest_version": 3,
  "permissions": []  // AI APIs work without extra permissions
}

// Extension code - Prompt API
async function usePromptAPI() {
  // Check if API is available
  const capabilities = await LanguageModel.availability();
  // 'unavailable' | 'downloadable' | 'downloading' | 'available'
  
  if (capabilities === 'unavailable') {
    console.log('AI not available on this device');
    return;
  }
  
  // Create a session
  const session = await LanguageModel.create({
    // Optional: system prompt
    systemPrompt: 'You are a helpful wiki editor assistant.'
  });
  
  // Simple prompt
  const result = await session.prompt('Summarize this text: ...');
  
  // Streaming response
  const stream = await session.promptStreaming('Write a summary...');
  for await (const chunk of stream) {
    console.log(chunk);
  }
  
  // Clone session for parallel requests
  const session2 = await session.clone();
  
  // Destroy session when done
  session.destroy();
}

// Summarizer API
async function useSummarizerAPI() {
  const summarizer = await Summarizer.create({
    type: 'key-points',  // 'key-points' | 'tl;dr' | 'teaser' | 'headline'
    format: 'markdown',  // 'plain-text' | 'markdown'
    length: 'short'      // 'short' | 'medium' | 'long'
  });
  
  const summary = await summarizer.summarize(longText);
}

// Translator API
async function useTranslatorAPI() {
  const translator = await Translator.create({
    sourceLanguage: 'en',
    targetLanguage: 'es'
  });
  
  const translation = await translator.translate('Hello world');
}

// Writer API (Origin Trial)
async function useWriterAPI() {
  const writer = await Writer.create({
    tone: 'formal',      // 'formal' | 'casual' | 'neutral'
    format: 'markdown',
    length: 'medium'
  });
  
  const content = await writer.write('Write about Azure DevOps wikis');
}

// Rewriter API (Origin Trial)
async function useRewriterAPI() {
  const rewriter = await Rewriter.create({
    tone: 'more-formal',
    format: 'as-is',
    length: 'as-is'
  });
  
  const improved = await rewriter.rewrite(originalText);
}
```

#### Chrome Flags for Local Testing

Enable these at `chrome://flags/`:
- `#optimization-guide-on-device-model` → Enabled BypassPerfRequirement
- `#prompt-api-for-gemini-nano` → Enabled
- `#prompt-api-for-gemini-nano-multimodal-input` → Enabled (for images/audio)
- `#summarization-api-for-gemini-nano` → Enabled
- `#translation-api-without-language-pack` → Enabled

Then trigger model download:
1. Open Chrome DevTools Console
2. Run: `await LanguageModel.create()`
3. Wait for model download (~22GB total space needed)

#### Supported Languages

- **Prompt API**: English, Japanese, Spanish (expanding)
- **Translator**: 20+ language pairs
- **Summarizer**: English (primary)

### Microsoft Edge AI Capabilities

#### Copilot Chat (NOT an API)

Edge's Copilot is a **sidebar feature**, not a developer API:
- Cloud-based (Microsoft servers)
- Requires Microsoft Entra account for enterprise features
- No JavaScript API exposed to extensions
- Cannot be integrated into custom extensions

#### Windows AI APIs (Native Only)

Microsoft provides AI APIs through **WinRT**, not web APIs:

| Feature | API Type | Availability |
|---------|----------|--------------|
| Phi Silica | WinRT (C#/C++) | Copilot+ PCs only |
| Text Recognition (OCR) | WinRT | Windows 10+ |
| AI Imaging | WinRT | Copilot+ PCs only |
| Content Moderation | WinRT | Windows 11+ |

**Key Limitation**: These APIs are NOT accessible from browser extensions or web pages. They require native Windows applications (UWP, WinUI, Win32).

#### WebNN API (Cross-Browser Standard)

WebNN is a **low-level ML inference API**, not an LLM API:

```javascript
// WebNN - for running ONNX models, not for LLM prompts
const context = await navigator.ml.createContext({
  powerPreference: 'high-performance',
  accelerated: true  // Use GPU/NPU
});

const builder = new MLGraphBuilder(context);
// ... build computation graph
const graph = await builder.build(outputs);

// Dispatch inference
context.dispatch(graph, inputs, outputs);
```

**Use Cases for WebNN**:
- Image classification
- Object detection
- Custom small models
- NOT suitable for LLM text generation

**Browser Support**:
- Chrome 138+: Enabled by default
- Edge 138+: Enabled by default (shares Chromium backend)
- Firefox: Not implemented
- Safari: Not implemented

### Comparison Summary

| Feature | Chrome AI | Edge AI | WebNN |
|---------|-----------|---------|-------|
| **LLM Prompts** | ✅ Prompt API | ❌ No API | ❌ Not designed for LLM |
| **Summarization** | ✅ Summarizer API | ❌ Copilot only | ❌ No |
| **Translation** | ✅ Translator API | ❌ No | ❌ No |
| **Writing Assistance** | ✅ Writer/Rewriter | ❌ No | ❌ No |
| **Custom ML Models** | Via WebNN | Via WebNN | ✅ Primary use |
| **Extension Support** | ✅ Full support | ❌ None | ✅ Via web workers |
| **Cost** | **FREE** | N/A | **FREE** |
| **Local/Cloud** | 100% Local | Cloud | Local |
| **Privacy** | High (on-device) | Low (cloud) | High (on-device) |

## Recommended Approach

### Primary: Chrome Prompt API for Extensions

Chrome's built-in AI is the **only viable option** for browser extension AI integration:

1. **Free to use** - No API costs, no quotas
2. **Privacy-preserving** - All processing on user's device
3. **Extension-ready** - Stable in Chrome 138+ for extensions
4. **Multiple APIs** - Prompt, Summarize, Translate, Write, Rewrite
5. **TypeScript support** - `@types/dom-chromium-ai` npm package available

### Implementation Considerations

#### Feature Detection Pattern

```typescript
// Safe feature detection for Chrome AI APIs
async function checkAIAvailability(): Promise<{
  prompt: boolean;
  summarizer: boolean;
  translator: boolean;
}> {
  return {
    prompt: 'LanguageModel' in globalThis && 
            (await LanguageModel.availability()) !== 'unavailable',
    summarizer: 'Summarizer' in globalThis &&
                (await Summarizer.availability()) !== 'unavailable',
    translator: 'Translator' in globalThis &&
                (await Translator.availability()) !== 'unavailable'
  };
}
```

#### Graceful Degradation

Since AI features require specific hardware:

```typescript
async function summarizeText(text: string): Promise<string> {
  // Try Chrome AI first
  if ('Summarizer' in globalThis) {
    const availability = await Summarizer.availability();
    if (availability === 'available') {
      const summarizer = await Summarizer.create();
      return await summarizer.summarize(text);
    }
  }
  
  // Fallback: simple extraction (first N sentences)
  return text.split('.').slice(0, 3).join('.') + '.';
}
```

#### Cross-Browser Consideration

- **Chrome/Chromium**: Full AI API support
- **Edge**: Same Chromium engine, but AI APIs may lag behind Chrome
- **Firefox/Safari**: No built-in AI APIs, would need cloud fallback

### Potential Use Cases for ADO Wiki Editor

1. **Smart Summarization**: Summarize long wiki pages for TOC previews
2. **Writing Assistance**: Help users draft wiki content
3. **Translation**: Translate wiki content to other languages
4. **Grammar/Proofreading**: Check and improve text quality
5. **Content Generation**: Generate placeholder content or documentation

### Limitations to Consider

1. **Hardware Requirements**: Not all users have capable hardware
2. **Model Size**: 22GB storage needed for full model
3. **Chrome Only**: No Firefox/Safari support
4. **English-focused**: Best results in English
5. **Context Limits**: Gemini Nano has smaller context than cloud models

## Advanced Implementation Patterns

### Session Management Best Practices

#### Token Quota Tracking

```typescript
// Monitor session usage to avoid exceeding limits
class AISessionManager {
  private session: LanguageModelSession | null = null;
  
  async getSession(): Promise<LanguageModelSession> {
    if (!this.session) {
      this.session = await LanguageModel.create({
        systemPrompt: 'You are a wiki editing assistant.'
      });
    }
    return this.session;
  }
  
  getQuotaStatus(): { used: number; total: number; remaining: number } {
    if (!this.session) {
      return { used: 0, total: 0, remaining: 0 };
    }
    const used = this.session.inputUsage;
    const total = this.session.inputQuota;
    return { used, total, remaining: total - used };
  }
  
  // Check if we have enough quota for a prompt
  async hasQuotaFor(prompt: string): Promise<boolean> {
    if (!this.session) return true;
    const usage = await this.session.measureInputUsage(prompt);
    return usage <= (this.session.inputQuota - this.session.inputUsage);
  }
  
  // Reset session when quota is low
  async resetIfNeeded(): Promise<void> {
    const { remaining, total } = this.getQuotaStatus();
    if (remaining < total * 0.1) { // Less than 10% remaining
      this.session?.destroy();
      this.session = null;
    }
  }
}
```

#### Session Cloning for Parallel Operations

```typescript
// Clone sessions for parallel requests without losing context
class ParallelAIProcessor {
  private mainSession: LanguageModelSession | null = null;
  
  async initialize(): Promise<void> {
    this.mainSession = await LanguageModel.create({
      systemPrompt: 'You are a wiki editing assistant specializing in Azure DevOps.'
    });
  }
  
  async processMultiple(prompts: string[]): Promise<string[]> {
    if (!this.mainSession) await this.initialize();
    
    // Clone session for each parallel request
    const sessions = await Promise.all(
      prompts.map(() => this.mainSession!.clone())
    );
    
    try {
      // Process all prompts in parallel
      const results = await Promise.all(
        sessions.map((session, i) => session.prompt(prompts[i]))
      );
      return results;
    } finally {
      // Clean up cloned sessions
      sessions.forEach(s => s.destroy());
    }
  }
}
```

#### Session Persistence (localStorage)

```typescript
// Restore conversation history across browser sessions
interface SessionData {
  uuid: string;
  initialPrompts: Array<{ role: string; content: string }>;
  topK: number;
  temperature: number;
}

class PersistentSession {
  private sessionId: string;
  private session: LanguageModelSession | null = null;
  
  constructor(sessionId: string) {
    this.sessionId = sessionId;
  }
  
  private loadSessionData(): SessionData | null {
    try {
      const stored = localStorage.getItem(`ai-session-${this.sessionId}`);
      return stored ? JSON.parse(stored) : null;
    } catch {
      return null;
    }
  }
  
  private saveSessionData(data: SessionData): void {
    localStorage.setItem(`ai-session-${this.sessionId}`, JSON.stringify(data));
  }
  
  async restore(): Promise<LanguageModelSession> {
    const data = this.loadSessionData();
    const params = await LanguageModel.params();
    
    const sessionConfig = data || {
      uuid: this.sessionId,
      initialPrompts: [
        { role: 'system', content: 'You are a wiki editing assistant.' }
      ],
      topK: params.defaultTopK,
      temperature: params.defaultTemperature
    };
    
    this.session = await LanguageModel.create(sessionConfig);
    return this.session;
  }
  
  async prompt(userPrompt: string): Promise<string> {
    if (!this.session) await this.restore();
    
    const result = await this.session!.prompt(userPrompt);
    
    // Save conversation history
    const data = this.loadSessionData()!;
    data.initialPrompts.push(
      { role: 'user', content: userPrompt },
      { role: 'assistant', content: result }
    );
    this.saveSessionData(data);
    
    return result;
  }
}
```

### Abort Controller Pattern

```typescript
// Allow users to cancel long-running AI operations
class CancellableAI {
  private controller: AbortController | null = null;
  
  async promptWithCancel(
    session: LanguageModelSession,
    prompt: string,
    onChunk?: (text: string) => void
  ): Promise<string> {
    this.controller = new AbortController();
    
    try {
      if (onChunk) {
        // Streaming with cancel support
        const stream = session.promptStreaming(prompt, {
          signal: this.controller.signal
        });
        let fullResponse = '';
        for await (const chunk of stream) {
          fullResponse = chunk; // Chunks are cumulative
          onChunk(chunk);
        }
        return fullResponse;
      } else {
        // Non-streaming with cancel support
        return await session.prompt(prompt, {
          signal: this.controller.signal
        });
      }
    } catch (err) {
      if (err.name === 'AbortError') {
        return ''; // Cancelled by user
      }
      throw err;
    } finally {
      this.controller = null;
    }
  }
  
  cancel(): void {
    this.controller?.abort();
  }
}
```

### Structured Output with JSON Schema

```typescript
// Enforce specific response formats
async function classifyContent(
  session: LanguageModelSession,
  content: string
): Promise<{ category: string; confidence: number }> {
  const schema = {
    type: 'object',
    properties: {
      category: {
        type: 'string',
        enum: ['documentation', 'tutorial', 'reference', 'changelog', 'other']
      },
      confidence: {
        type: 'number',
        minimum: 0,
        maximum: 1
      }
    },
    required: ['category', 'confidence']
  };
  
  const result = await session.prompt(
    `Classify this wiki content:\n\n${content}`,
    { responseConstraint: schema }
  );
  
  return JSON.parse(result);
}
```

## Fallback Strategies

### Strategy 1: Simple Local Fallback (No Cloud)

```typescript
// Fallback to basic text processing when AI unavailable
class AIWithLocalFallback {
  private aiAvailable = false;
  private session: LanguageModelSession | null = null;
  
  async initialize(): Promise<void> {
    if ('LanguageModel' in globalThis) {
      const availability = await LanguageModel.availability();
      this.aiAvailable = availability !== 'unavailable';
      if (this.aiAvailable && availability === 'available') {
        this.session = await LanguageModel.create();
      }
    }
  }
  
  async summarize(text: string): Promise<string> {
    if (this.session) {
      try {
        return await this.session.prompt(
          `Summarize this text in 2-3 sentences:\n\n${text}`
        );
      } catch {
        // Fall through to local fallback
      }
    }
    
    // Local fallback: extract first sentences
    const sentences = text.match(/[^.!?]+[.!?]+/g) || [];
    return sentences.slice(0, 3).join(' ').trim() || text.slice(0, 200) + '...';
  }
  
  async improveWriting(text: string): Promise<string> {
    if (this.session) {
      try {
        return await this.session.prompt(
          `Improve this text for clarity and grammar:\n\n${text}`
        );
      } catch {
        // Fall through to local fallback
      }
    }
    
    // Local fallback: return original (no improvement possible)
    return text;
  }
  
  isAIAvailable(): boolean {
    return this.aiAvailable && this.session !== null;
  }
}
```

### Strategy 2: Firebase AI Logic Hybrid (Cloud Fallback)

```typescript
// Use Firebase for cloud fallback when built-in AI unavailable
import { initializeApp } from 'firebase/app';
import { getAI, getGenerativeModel } from 'firebase/ai';

class HybridAI {
  private builtInSession: LanguageModelSession | null = null;
  private firebaseModel: any = null;
  private useBuiltIn = false;
  
  async initialize(firebaseConfig: object): Promise<void> {
    // Try built-in AI first
    if ('LanguageModel' in globalThis) {
      const availability = await LanguageModel.availability();
      if (availability === 'available') {
        this.builtInSession = await LanguageModel.create();
        this.useBuiltIn = true;
        console.log('Using built-in AI (Gemini Nano)');
        return;
      }
    }
    
    // Fall back to Firebase
    const app = initializeApp(firebaseConfig);
    const ai = getAI(app);
    this.firebaseModel = getGenerativeModel(ai, { 
      mode: 'prefer_on_device' // Will use cloud if on-device unavailable
    });
    console.log('Using Firebase AI Logic (cloud fallback)');
  }
  
  async prompt(text: string): Promise<string> {
    if (this.useBuiltIn && this.builtInSession) {
      return await this.builtInSession.prompt(text);
    }
    
    if (this.firebaseModel) {
      const result = await this.firebaseModel.generateContent(text);
      return result.response.text();
    }
    
    throw new Error('No AI backend available');
  }
  
  async *promptStreaming(text: string): AsyncGenerator<string> {
    if (this.useBuiltIn && this.builtInSession) {
      const stream = this.builtInSession.promptStreaming(text);
      for await (const chunk of stream) {
        yield chunk;
      }
      return;
    }
    
    if (this.firebaseModel) {
      const result = await this.firebaseModel.generateContentStream(text);
      for await (const chunk of result.stream) {
        yield chunk.text();
      }
      return;
    }
    
    throw new Error('No AI backend available');
  }
  
  getBackend(): 'built-in' | 'firebase' | 'none' {
    if (this.useBuiltIn) return 'built-in';
    if (this.firebaseModel) return 'firebase';
    return 'none';
  }
}
```

### Strategy 3: Graceful Feature Degradation UI

```typescript
// Manage UI based on AI availability
interface AIFeatureState {
  summarization: 'available' | 'limited' | 'unavailable';
  translation: 'available' | 'limited' | 'unavailable';
  writing: 'available' | 'limited' | 'unavailable';
  proofreading: 'available' | 'limited' | 'unavailable';
}

class AIFeatureManager {
  private state: AIFeatureState = {
    summarization: 'unavailable',
    translation: 'unavailable',
    writing: 'unavailable',
    proofreading: 'unavailable'
  };
  
  async detectCapabilities(): Promise<AIFeatureState> {
    // Check Summarizer
    if ('Summarizer' in globalThis) {
      const avail = await Summarizer.availability();
      this.state.summarization = avail === 'available' ? 'available' : 
                                  avail === 'downloadable' ? 'limited' : 'unavailable';
    }
    
    // Check Translator
    if ('Translator' in globalThis) {
      const avail = await Translator.availability();
      this.state.translation = avail === 'available' ? 'available' : 
                                avail === 'downloadable' ? 'limited' : 'unavailable';
    }
    
    // Check Writer (Origin Trial)
    if ('Writer' in globalThis) {
      try {
        const avail = await Writer.availability();
        this.state.writing = avail === 'available' ? 'available' : 
                              avail === 'downloadable' ? 'limited' : 'unavailable';
      } catch {
        this.state.writing = 'unavailable';
      }
    }
    
    // Check Proofreader (Origin Trial)
    if ('Proofreader' in globalThis) {
      try {
        const avail = await Proofreader.availability();
        this.state.proofreading = avail === 'available' ? 'available' : 
                                   avail === 'downloadable' ? 'limited' : 'unavailable';
      } catch {
        this.state.proofreading = 'unavailable';
      }
    }
    
    return this.state;
  }
  
  getUIConfig(): {
    showSummarizeButton: boolean;
    showTranslateButton: boolean;
    showWriteAssistButton: boolean;
    showProofreadButton: boolean;
    aiStatusMessage: string;
  } {
    const availableCount = Object.values(this.state)
      .filter(s => s === 'available').length;
    
    return {
      showSummarizeButton: this.state.summarization !== 'unavailable',
      showTranslateButton: this.state.translation !== 'unavailable',
      showWriteAssistButton: this.state.writing !== 'unavailable',
      showProofreadButton: this.state.proofreading !== 'unavailable',
      aiStatusMessage: availableCount === 0 
        ? 'AI features unavailable on this device'
        : availableCount === 4 
          ? 'All AI features ready'
          : `${availableCount} AI features available`
    };
  }
}
```

### Model Download Progress Handling

```typescript
// Inform users during model download
async function createSessionWithProgress(
  onProgress: (percent: number) => void,
  onReady: () => void
): Promise<LanguageModelSession> {
  const availability = await LanguageModel.availability();
  
  if (availability === 'unavailable') {
    throw new Error('AI not supported on this device');
  }
  
  return await LanguageModel.create({
    monitor(m) {
      m.addEventListener('downloadprogress', (e) => {
        const percent = e.loaded * 100;
        onProgress(percent);
        if (percent >= 100) {
          onReady();
        }
      });
    }
  });
}

// Usage
const session = await createSessionWithProgress(
  (percent) => {
    progressBar.style.width = `${percent}%`;
    statusText.textContent = `Downloading AI model: ${percent.toFixed(0)}%`;
  },
  () => {
    statusText.textContent = 'AI ready!';
    progressBar.style.display = 'none';
  }
);
```

## Error Handling Patterns

```typescript
// Comprehensive error handling for AI operations
type AIError = 
  | { type: 'unavailable'; message: string }
  | { type: 'quota_exceeded'; message: string }
  | { type: 'aborted'; message: string }
  | { type: 'network'; message: string }
  | { type: 'unknown'; message: string; originalError: Error };

async function safeAIPrompt(
  session: LanguageModelSession,
  prompt: string,
  signal?: AbortSignal
): Promise<{ success: true; result: string } | { success: false; error: AIError }> {
  try {
    const result = await session.prompt(prompt, { signal });
    return { success: true, result };
  } catch (err) {
    if (err.name === 'AbortError') {
      return { 
        success: false, 
        error: { type: 'aborted', message: 'Operation cancelled' } 
      };
    }
    
    if (err.name === 'QuotaExceededError') {
      return { 
        success: false, 
        error: { type: 'quota_exceeded', message: 'Session token limit reached' } 
      };
    }
    
    if (err.name === 'NotSupportedError') {
      return { 
        success: false, 
        error: { type: 'unavailable', message: 'AI feature not supported' } 
      };
    }
    
    if (err.name === 'NetworkError') {
      return { 
        success: false, 
        error: { type: 'network', message: 'Network error during AI operation' } 
      };
    }
    
    return { 
      success: false, 
      error: { type: 'unknown', message: err.message, originalError: err } 
    };
  }
}
```

## Implementation Guidance

- **Objectives**: Add optional AI-powered features to ADO Wiki editor extension
- **Key Tasks**:
  1. Add feature detection for Chrome AI APIs
  2. Implement graceful degradation for unsupported browsers
  3. Add AI-powered summarization for TOC generation
  4. Consider optional writing assistance features
  5. Implement session management for conversation context
  6. Add abort controller for long operations
  7. Handle model download progress for UX
- **Dependencies**: Chrome 138+, `@types/dom-chromium-ai` for TypeScript
- **Optional**: Firebase AI Logic for cloud fallback
- **Success Criteria**: AI features work on capable hardware, extension works without AI on other browsers
