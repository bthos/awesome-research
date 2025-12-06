<!-- markdownlint-disable-file -->

# Task Research Notes: ADO Attachments & Milkdown Integration

## Research Executed

### ADO File Analysis

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wiki-restclient-content.es6.8_Wgng.min.js.download`
  - Wiki REST API client with `createAttachment` method
  - API endpoint: `{project}/_apis/wiki/wikis/{wikiIdentifier}/attachments/{name}`
  - API version: 5.2-preview.1

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wiki-editor-content.es6.mD8YC9.min.js.download`
  - `FileSource` class for reading file contents as base64
  - `AttachmentHelpers` module for file validation and markdown generation
  - `guidSuffixedFileName` generation pattern
  - `addAttachments` action creator

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wiki-view-common-content.es6.zwkPqF.min.js.download`
  - `AllowedAttachmentFileTypes` list

- `.inputs/examples/Editing Page 1 - Wiki - Overview_files/ms.vss-wiki-web.wiki-renderer-content.es6.cuMuSd.min.js.download`
  - Image URL transformer
  - `.attachments` path handling

### Milkdown Documentation

- #context7: `/milkdown/milkdown` - Upload plugin and image components
- #context7: `/milkdown/website` - Image block configuration

## Key Discoveries

### ADO Attachment System

#### File Storage Path
ADO Wiki stores attachments in a special `/.attachments/` folder at the wiki root:

```
/.attachments/
  image-name-{GUID}.png
  document-{GUID}.pdf
```

#### File Naming Convention
```javascript
// ADO generates a GUID-suffixed filename to prevent collisions
const baseName = file.name.substring(0, file.name.lastIndexOf("."));
const extension = getFileExtension(file.name);
file.guidSuffixedFileName = `${baseName}-${newGuid()}.${extension}`;
```

#### Markdown Output Format
```javascript
function getMarkdownText(files) {
  let text = "";
  for (const file of files) {
    let markdown = `[${file.name}](/.attachments/${encodeSpaces(file.guidSuffixedFileName)})`;
    // Prefix with ! for images
    if (file.type && file.type.indexOf("image/") === 0) {
      markdown = `!${markdown}`;
    }
    text += markdown;
  }
  return text;
}

function encodeSpaces(name) {
  return name.replace(/ /g, "%20");
}
```

#### REST API for Upload

```typescript
// Wiki/Clients/Wiki module
class WikiRestClient extends RestClientBase {
  async createAttachment(
    content: ArrayBuffer | Blob | string,  // File content (base64 or raw)
    project: string,                        // Project ID
    wikiIdentifier: string,                 // Wiki ID
    name: string,                           // Filename (guidSuffixedFileName)
    versionDescriptor?: WikiVersionDescriptor
  ): Promise<WikiAttachment> {
    return this.beginRequest({
      apiVersion: "5.2-preview.1",
      method: "PUT",
      routeTemplate: "{project}/_apis/wiki/wikis/{wikiIdentifier}/attachments/{name}",
      routeValues: { project, wikiIdentifier },
      customHeaders: { "Content-Type": "application/octet-stream" },
      queryParams: { name, versionDescriptor },
      body: content,
      isRawData: true,
      returnRawResponse: true
    });
  }
}
```

#### Allowed File Types

```javascript
const AllowedAttachmentFileTypes = [
  ".CS", ".CSV", ".DOC", ".DOCX", ".GIF", ".GZ", ".HTM", ".HTML",
  ".ICO", ".JPEG", ".JPG", ".JSON", ".LYR", ".MD", ".MOV", ".MP4",
  ".MPP", ".MSG", ".PDF", ".PNG", ".PPT", ".PPTX", ".PS1", ".RAR",
  ".RDP", ".SQL", ".TXT", ".VSD", ".VSDX", ".XLS", ".XLSX", ".XML",
  ".ZIP", ".SVG" // SVG requires feature flag
];
```

#### File Size Limit

```javascript
const MAX_ATTACHMENT_FILE_SIZE = 18874368; // 18 MB
```

#### FileSource Class (File Reading)

```typescript
class FileSource {
  static createAttachment(file: File): IAttachment {
    return {
      file,
      base64Content: undefined,
      error: file.size > MAX_ATTACHMENT_FILE_SIZE 
        ? new Error("Exceeded size limit") 
        : undefined
    };
  }
  
  async readContent(attachment: IAttachment): Promise<IAttachment> {
    if (attachment.error) return attachment;
    
    const reader = new FileReader();
    return new Promise(resolve => {
      reader.onloadend = () => {
        const result = reader.result as string;
        const commaIndex = result.indexOf(",");
        if (commaIndex >= 0) {
          let base64 = result.substr(commaIndex + 1);
          // Handle edge case for base64 padding
          if (base64.substr(0, 2) === "//" && base64.length % 4 === 2) {
            base64 = base64.substr(2);
          }
          attachment.base64Content = base64;
        }
        resolve(attachment);
      };
      reader.readAsDataURL(attachment.blob);
    });
  }
}
```

#### Image URL Resolution

```javascript
// ImageTransformer module - resolves relative paths to full URLs
function getImageUrl(src, pageContext, wikiRootPath, projectId, repositoryId, versionDescriptor) {
  // Check if it's a .attachments path
  const attachmentsPath = "/.attachments";
  const isAttachment = src.startsWith("/") || src.startsWith(attachmentsPath.substr(1));
  
  // For attachments, construct the path relative to wiki root
  const basePath = isAttachment ? wikiRootPath : getCurrentPagePath();
  const normalizedSrc = src.startsWith("/") ? src.substr(1) : src;
  const fullPath = combinePaths(basePath, normalizedSrc);
  
  // Generate Git file content URL
  return getGitFileContentUrl(pageContext, projectId, repositoryId, versionDescriptor, fullPath, true);
}
```

### Milkdown Upload System

#### Upload Plugin Configuration

```typescript
import { upload, uploadConfig, Uploader } from '@milkdown/kit/plugin/upload';
import type { Node } from '@milkdown/kit/prose/model';

const uploader: Uploader = async (files, schema) => {
  const images: File[] = [];

  for (let i = 0; i < files.length; i++) {
    const file = files.item(i);
    if (!file || !file.type.includes('image')) continue;
    images.push(file);
  }

  const nodes: Node[] = await Promise.all(
    images.map(async (image) => {
      const src = await uploadToServer(image); // Custom upload function
      return schema.nodes.image.createAndFill({ src, alt: image.name }) as Node;
    })
  );

  return nodes;
};

Editor.make()
  .config((ctx) => {
    ctx.update(uploadConfig.key, (prev) => ({
      ...prev,
      uploader,
    }));
  })
  .use(upload)
  .create();
```

#### Image Block Component

```typescript
import { imageBlockComponent, imageBlockConfig } from '@milkdown/components/image-block';

Editor.make()
  .config((ctx) => {
    ctx.update(imageBlockConfig.key, (defaultConfig) => ({
      ...defaultConfig,
      onUpload: async (file: File) => {
        const url = await uploadToServer(file);
        return url;
      },
      imageIcon: '���️',
      uploadButton: 'Upload Image',
      uploadPlaceholderText: 'or paste an image URL',
      captionPlaceholderText: 'Add a caption',
    }));
  })
  .use(imageBlockComponent)
  .create();
```

#### Inline Image Component

```typescript
import { inlineImageConfig } from '@milkdown/components/image-inline';

ctx.update(inlineImageConfig.key, (defaultConfig) => ({
  ...defaultConfig,
  onUpload: async (file: File) => {
    const url = await uploadToServer(file);
    return url;
  },
  proxyDomURL: (originalURL: string) => {
    // Transform relative URLs to full URLs
    return transformToFullUrl(originalURL);
  },
}));
```

## Recommended Integration Approach

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Milkdown Editor                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Image Block / Upload Plugin                         │   │
│  │  onUpload: (file) => Promise<string>                 │   │
│  └────────────────────────┬────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              ADO Attachment Upload Service                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  1. Validate file type and size                      │   │
│  │  2. Generate GUID-suffixed filename                  │   │
│  │  3. Read file as base64                              │   │
│  │  4. Call Wiki REST API                               │   │
│  │  5. Return /.attachments/{filename}                  │   │
│  └────────────────────────┬────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              ADO Wiki REST API                              │
│  PUT {project}/_apis/wiki/wikis/{wikiId}/attachments/{name} │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Plan

#### 1. ADO Attachment Service (`src/services/attachment-service.ts`)

```typescript
import { v4 as uuidv4 } from 'uuid';

interface IAttachment {
  file: File;
  guidSuffixedFileName: string;
  base64Content?: string;
  error?: Error;
}

interface IWikiContext {
  projectId: string;
  wikiId: string;
  wikiVersion?: string;
  accessToken: string;
}

// Allowed file types (matching ADO's official list)
const ALLOWED_ATTACHMENT_TYPES = [
  '.CS', '.CSV', '.DOC', '.DOCX', '.GIF', '.GZ', '.HTM', '.HTML',
  '.ICO', '.JPEG', '.JPG', '.JSON', '.LYR', '.MD', '.MOV', '.MP4',
  '.MPP', '.MSG', '.PDF', '.PNG', '.PPT', '.PPTX', '.PS1', '.RAR',
  '.RDP', '.SQL', '.TXT', '.VSD', '.VSDX', '.XLS', '.XLSX', '.XML',
  '.ZIP', '.SVG'
];

const MAX_FILE_SIZE = 18874368; // 18 MB

export class AdoAttachmentService {
  constructor(private wikiContext: IWikiContext) {}

  // Validate file is allowed
  validateFile(file: File): { valid: boolean; error?: string } {
    const ext = '.' + file.name.split('.').pop()?.toUpperCase();
    
    if (!ALLOWED_ATTACHMENT_TYPES.includes(ext)) {
      return { valid: false, error: `File type ${ext} is not supported` };
    }
    
    if (file.size > MAX_FILE_SIZE) {
      return { valid: false, error: `File exceeds maximum size of 18MB` };
    }
    
    return { valid: true };
  }

  // Generate ADO-style GUID-suffixed filename
  generateGuidSuffixedFilename(originalName: string): string {
    const lastDotIndex = originalName.lastIndexOf('.');
    const baseName = lastDotIndex > 0 ? originalName.substring(0, lastDotIndex) : originalName;
    const extension = lastDotIndex > 0 ? originalName.substring(lastDotIndex + 1) : '';
    const guid = uuidv4().replace(/-/g, '');
    return `${baseName}-${guid}.${extension}`;
  }

  // Read file as base64
  async readFileAsBase64(file: File): Promise<string> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onloadend = () => {
        const result = reader.result as string;
        const commaIndex = result.indexOf(',');
        if (commaIndex >= 0) {
          resolve(result.substr(commaIndex + 1));
        } else {
          reject(new Error('Failed to read file as base64'));
        }
      };
      reader.onerror = () => reject(reader.error);
      reader.readAsDataURL(file);
    });
  }

  // Upload attachment to ADO Wiki
  async uploadAttachment(file: File): Promise<string> {
    const validation = this.validateFile(file);
    if (!validation.valid) {
      throw new Error(validation.error);
    }

    const guidSuffixedFileName = this.generateGuidSuffixedFilename(file.name);
    const base64Content = await this.readFileAsBase64(file);
    
    // Convert base64 to ArrayBuffer for upload
    const binaryString = atob(base64Content);
    const bytes = new Uint8Array(binaryString.length);
    for (let i = 0; i < binaryString.length; i++) {
      bytes[i] = binaryString.charCodeAt(i);
    }

    const url = `${this.getApiBaseUrl()}/_apis/wiki/wikis/${this.wikiContext.wikiId}/attachments/${encodeURIComponent(guidSuffixedFileName)}?api-version=5.2-preview.1`;
    
    const response = await fetch(url, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/octet-stream',
        'Authorization': `Bearer ${this.wikiContext.accessToken}`,
      },
      body: bytes.buffer,
    });

    if (!response.ok) {
      throw new Error(`Failed to upload attachment: ${response.statusText}`);
    }

    // Return the markdown-ready path
    return `/.attachments/${encodeURIComponent(guidSuffixedFileName).replace(/%20/g, '%20')}`;
  }

  private getApiBaseUrl(): string {
    // Get ADO organization URL from page context
    return `https://dev.azure.com/${this.wikiContext.projectId}`;
  }
}
```

#### 2. Milkdown Upload Integration (`src/plugins/attachment-upload.ts`)

```typescript
import { upload, uploadConfig, Uploader } from '@milkdown/kit/plugin/upload';
import { imageBlockConfig } from '@milkdown/components/image-block';
import { inlineImageConfig } from '@milkdown/components/image-inline';
import { Ctx } from '@milkdown/kit/ctx';
import { AdoAttachmentService } from '../services/attachment-service';

export function configureAttachmentUpload(ctx: Ctx, attachmentService: AdoAttachmentService) {
  // Configure drag-and-drop upload plugin
  const uploader: Uploader = async (files, schema) => {
    const nodes = [];

    for (let i = 0; i < files.length; i++) {
      const file = files.item(i);
      if (!file) continue;

      try {
        const attachmentPath = await attachmentService.uploadAttachment(file);
        
        if (file.type.startsWith('image/')) {
          // Create image node
          const node = schema.nodes.image.createAndFill({
            src: attachmentPath,
            alt: file.name,
          });
          if (node) nodes.push(node);
        } else {
          // Create link node for non-images
          const node = schema.nodes.paragraph.create({}, [
            schema.text(`[${file.name}](${attachmentPath})`)
          ]);
          if (node) nodes.push(node);
        }
      } catch (error) {
        console.error('Upload failed:', error);
        // Could show error notification to user
      }
    }

    return nodes;
  };

  ctx.update(uploadConfig.key, (prev) => ({
    ...prev,
    uploader,
  }));

  // Configure image block component
  ctx.update(imageBlockConfig.key, (defaultConfig) => ({
    ...defaultConfig,
    onUpload: async (file: File) => {
      return await attachmentService.uploadAttachment(file);
    },
    uploadPlaceholderText: 'Paste URL or drop image',
    captionPlaceholderText: 'Add caption',
  }));

  // Configure inline image component
  ctx.update(inlineImageConfig.key, (defaultConfig) => ({
    ...defaultConfig,
    onUpload: async (file: File) => {
      return await attachmentService.uploadAttachment(file);
    },
    proxyDomURL: (originalURL: string) => {
      // Transform /.attachments/ paths to full Git URLs for preview
      if (originalURL.startsWith('/.attachments/')) {
        return transformToGitUrl(originalURL, attachmentService.wikiContext);
      }
      return originalURL;
    },
  }));
}

function transformToGitUrl(attachmentPath: string, wikiContext: IWikiContext): string {
  // Transform relative attachment path to full Git file URL
  // This is needed for preview/display in the editor
  const { projectId, wikiId, wikiVersion } = wikiContext;
  const encodedPath = encodeURIComponent(attachmentPath);
  return `https://dev.azure.com/${projectId}/_apis/git/repositories/${wikiId}/items?path=${encodedPath}&versionDescriptor=${wikiVersion}`;
}
```

#### 3. Editor Integration (`src/editor-bundle.ts`)

```typescript
import { upload } from '@milkdown/kit/plugin/upload';
import { imageBlockComponent } from '@milkdown/components/image-block';
import { AdoAttachmentService } from './services/attachment-service';
import { configureAttachmentUpload } from './plugins/attachment-upload';

// ... existing imports ...

export function createEditor(element: HTMLElement, wikiContext: IWikiContext) {
  const attachmentService = new AdoAttachmentService(wikiContext);
  
  return Editor.make()
    .config((ctx) => {
      ctx.set(rootCtx, element);
      configureAttachmentUpload(ctx, attachmentService);
    })
    .use(commonmark)
    .use(gfm)
    .use(upload)              // Enable drag-and-drop upload
    .use(imageBlockComponent) // Enable image block UI
    .use(adoSyntaxPlugin)
    .use(toolbarPlugin)
    .create();
}
```

### Browser Extension Considerations

Since this runs as a browser extension, we can:

1. **Intercept ADO's authentication**: Get the access token from the existing ADO session
2. **Access Wiki context**: Extract project ID, wiki ID from the page URL
3. **Use ADO's existing API**: The REST API is already authenticated via the browser session

```typescript
// Get wiki context from ADO page
function getWikiContextFromPage(): IWikiContext {
  // Extract from ADO's page context (window.__vssPageContext)
  const pageContext = (window as any).__vssPageContext || {};
  const webContext = pageContext.webContext || {};
  
  return {
    projectId: webContext.project?.id,
    wikiId: extractWikiIdFromUrl(),
    wikiVersion: pageContext.navigation?.routeValues?.wikiVersion,
    accessToken: getAccessTokenFromSession(),
  };
}

function extractWikiIdFromUrl(): string {
  // URL pattern: /_wiki/wikis/{wikiId}/...
  const match = window.location.pathname.match(/\/_wiki\/wikis\/([^/]+)/);
  return match ? match[1] : '';
}

function getAccessTokenFromSession(): string {
  // ADO stores auth in session - can use existing cookies/session
  // For API calls, the browser's session cookies will authenticate
  return ''; // Use fetch with credentials: 'include' instead
}
```

## Implementation Guidance

- **Objectives**: Enable file upload and attachment support matching ADO's native behavior
- **Key Tasks**:
  1. Create `AdoAttachmentService` class for file validation and upload
  2. Configure Milkdown's `upload` plugin with custom uploader
  3. Configure `imageBlockComponent` for image UI
  4. Extract wiki context from ADO page
  5. Handle image URL transformation for preview
- **Dependencies**:
  - `@milkdown/kit/plugin/upload`
  - `@milkdown/components/image-block`
  - UUID library for GUID generation (or use browser's `crypto.randomUUID()`)
- **Success Criteria**:
  - Drag-and-drop file upload works
  - Files are uploaded to `/.attachments/` folder
  - Images render correctly with `![alt](/.attachments/filename)` syntax
  - Non-image files create download links
  - File size and type validation matches ADO's rules
