# Awesome List Format Instructions

This document defines the "awesome list" format rules for the **awesome-research** repository.

## Repository Structure

```
awesome-research/
├── README.md                    # Main index/awesome list
├── .github/
│   ├── copilot-instructions.md  # AI agent instructions
│   └── awesome-list-instructions.md  # This file
└── research/                  # Research documents folder
    └── YYYYMMDD-topic-research.md
```

## README.md (Main Index)

The README.md serves as the curated "awesome list" index. Format:

```markdown
# awesome-research
A collection of research results covering different topics

## Contents

- [Category Name](#category-name)

## Category Name

- [Research Title](research/YYYYMMDD-topic-research.md) - Brief one-line description
```

**Index Guidelines:**

- Group research documents by relevant categories
- Use descriptive, concise one-line descriptions
- Keep entries sorted chronologically within categories (newest first) or alphabetically
- Link directly to the research document in `research/` folder

## Research Document Naming Convention

**Pattern:** `YYYYMMDD-descriptive-topic-research.md`

- `YYYYMMDD` - Date the research was created (e.g., `20251130`)
- `descriptive-topic` - Kebab-case topic description (e.g., `browser-ai-api`, `ado-attachment-milkdown-integration`)
- Always suffix with `-research.md`

**Examples:**

- `20251130-ado-attachment-milkdown-integration-research.md`
- `20251130-browser-ai-api-research.md`
- `20250530-toc-tosp-widget-styling-research.md`

## Research Document Structure

Each research document MUST follow this template:

```markdown
<!-- markdownlint-disable-file -->

# Task Research Notes: [Topic Title]

## Research Executed

### [Source Category 1]

- `source/path/or/url`
  - Key finding 1
  - Key finding 2

### [Source Category 2]

- Source description
  - Finding details

## Key Discoveries

### [Discovery Area 1]

#### [Sub-topic]

Detailed explanation with code examples if applicable:

\`\`\`language
// Code example
\`\`\`

### [Discovery Area 2]

Continue with structured findings...

## Implementation Notes

(Optional) Technical implementation details, patterns, or recommendations.

## References

(Optional) External links, documentation, or related resources.

## Conclusion

(Optional) Summary of findings and recommendations.
```

**Document Guidelines:**

- Start with `<!-- markdownlint-disable-file -->` comment
- Use `# Task Research Notes: [Topic]` as the main heading
- Structure content hierarchically with clear headings
- Include code examples in fenced code blocks with language specifiers
- Use tables for comparative data or status matrices
- Document sources analyzed in "Research Executed" section
- Present findings in "Key Discoveries" section with sub-sections

## AI Assistant Instructions

### When Creating New Research

1. **Generate filename** using pattern: `YYYYMMDD-topic-research.md`
   - Use current date for YYYYMMDD
   - Use descriptive kebab-case topic name

2. **Create document** in `research/` folder following the template structure

3. **Update README.md** to add the new research entry:
   - Add appropriate category if it doesn't exist
   - Add entry with link and brief description
   - Maintain sorting order

### When Modifying Existing Research

1. Preserve the document structure and formatting conventions
2. Add new sections following the established heading hierarchy
3. Maintain code block language specifiers
4. Keep the `<!-- markdownlint-disable-file -->` comment at the top

### When Querying Research

1. Search by topic keywords in filenames
2. Reference specific sections using markdown heading anchors
3. Cross-reference related research documents when applicable

## Content Quality Standards

- **Clarity:** Use clear, technical language
- **Structure:** Maintain consistent heading hierarchy
- **Code:** Include relevant code examples with proper syntax highlighting
- **Sources:** Document all sources analyzed
- **Completeness:** Cover the topic thoroughly with actionable findings

## Markdown Formatting Standards

- Use ATX-style headers (`#`, `##`, `###`)
- Use fenced code blocks with language identifiers
- Use bullet lists for enumerations
- Use tables for comparative data
- Use inline code backticks for file paths, code references, and technical terms
- Maintain proper indentation in nested lists

## Workflow

### Adding New Research

```bash
# 1. Create research file
research/YYYYMMDD-topic-research.md

# 2. Update README.md with new entry
- [Topic Title](research/YYYYMMDD-topic-research.md) - Brief description

# 3. Commit with descriptive message
git commit -m "Add research: Topic Title"
```

### Research Document Lifecycle

1. **Draft:** Initial research notes and findings
2. **Complete:** Fully documented with all sections
3. **Updated:** Revised with new findings (add date annotations if significant)

## References

- [Awesome List Guidelines](https://github.com/sindresorhus/awesome/blob/main/awesome.md)
- [Markdown Guide](https://www.markdownguide.org/)
- [Conventional Commits](https://www.conventionalcommits.org/)
