---
description: AI rules derived by SpecStory from the project AI interaction history
globs: *
---

## PROJECT RULES, CODING STANDARDS, WORKFLOW GUIDELINES, REFERENCES, DOCUMENTATION STRUCTURES, AND BEST PRACTICES

This file serves as the central repository for all project-related rules, coding standards, workflow guidelines, references, documentation structures, and best practices. It is a "living" document, continuously updated and refined based on new user-AI interactions and project needs.

## PROJECT DESCRIPTION & GOALS

**awesome-research** is a curated collection of research documents covering various technical topics. Following the "awesome list" pattern, this repository serves as:

- A centralized knowledge base for research findings
- A reference collection for implementation patterns and best practices
- A living archive of technical investigations

## REPOSITORY STRUCTURE

```
awesome-research/
├── README.md                    # Main index/awesome list
├── .github/
│   ├── copilot-instructions.md  # AI agent instructions (this file)
│   └── awesome-list-instructions.md  # Awesome list format rules
└── research/                  # Research documents folder
    └── YYYYMMDD-topic-research.md
```

## AI ASSISTANT INSTRUCTIONS

The AI assistant is responsible for maintaining the "awesome list" format of the repository. 

**See [awesome-list-instructions.md](awesome-list-instructions.md) for detailed format rules, templates, and guidelines.**

Key responsibilities:

1. **Creating New Research:** Generate properly named files in `research/` folder and update README.md index
2. **Modifying Research:** Preserve document structure and formatting conventions
3. **Querying Research:** Search by topic keywords and cross-reference related documents

## REFERENCES

- [Awesome List Format Instructions](awesome-list-instructions.md) - Detailed format rules and templates
- [Awesome List Guidelines](https://github.com/sindresorhus/awesome/blob/main/awesome.md)
- [Markdown Guide](https://www.markdownguide.org/)
- [Conventional Commits](https://www.conventionalcommits.org/)