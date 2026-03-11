# GitHub Copilot Instructions

## Skills Library

This workspace contains a library of agent skills located at `skills/<skill-name>/SKILL.md`. **Before starting any task, proactively scan and reference all relevant skills.** Do not wait to be asked — discovering and applying the right skills is part of how you deliver high-quality, consistent results.

### How to Use Skills

1. **Before beginning any task**, review the list of available skills below and identify which ones apply.
2. **Read the full `SKILL.md`** for every relevant skill before taking action.
3. **Apply the guidance** from those skills throughout your response.
4. If a task spans multiple domains (e.g., a new feature needs requirements analysis, architecture, frontend and backend implementation, and testing), apply all relevant skills in sequence.

### Available Skills

| Skill | Path | When to Use |
|-------|------|-------------|
| `analyze-requirements` | `skills/analyze-requirements/SKILL.md` | Analyze requirements and decompose tasks before implementation begins. Use when given a task description, user story, feature request, change request, or project specification — regardless of apparent simplicity. |
| `api-design` | `skills/api-design/SKILL.md` | Design production-grade APIs that are consistent, versioned, and safe to evolve. Use before implementing any API endpoint, REST resource, GraphQL schema, or service interface. |
| `architecture-and-planning` | `skills/architecture-and-planning/SKILL.md` | Design scalable, maintainable system architecture and project planning before implementation. Use when starting a new project, major feature, or when making significant architectural decisions. |
| `authentication-and-authorization` | `skills/authentication-and-authorization/SKILL.md` | Implement production-hardened, standards-compliant authentication and authorization. Use when building login systems, OAuth flows, JWT handling, RBAC, or any access control mechanism. |
| `backend-service-development` | `skills/backend-service-development/SKILL.md` | Build production-grade backend API services that are reliable, maintainable, observable, and secure. Use when developing server-side APIs, microservices, or backend business logic. |
| `brand-guidelines` | `skills/brand-guidelines/SKILL.md` | Applies Anthropic's official brand colors and typography to any artifact. Use when brand colors, style guidelines, visual formatting, or company design standards apply. |
| `canvas-design` | `skills/canvas-design/SKILL.md` | Create beautiful visual art in .png and .pdf documents using design philosophy. Use when the user asks to create a poster, piece of art, design, or other static piece. |
| `claude-api` | `skills/claude-api/SKILL.md` | Build apps with the Claude API or Anthropic SDK. Use when code imports `anthropic`/`@anthropic-ai/sdk`/`claude_agent_sdk`, or when the user asks to use the Claude API or Anthropic SDKs. |
| `code-quality-and-review` | `skills/code-quality-and-review/SKILL.md` | Maintain code correctness, readability, and safety. Use when reviewing code, performing refactoring, enforcing code quality standards, or conducting technical debt analysis. |
| `data-management` | `skills/data-management/SKILL.md` | Design and manage reliable, performant, and safe data persistence. Use when working with databases, data models, schema design, migrations, caching strategies, or data lifecycle management. |
| `deployment-and-release` | `skills/deployment-and-release/SKILL.md` | Ship software to production reliably, safely, and repeatably. Use when deploying applications, configuring CI/CD pipelines, managing release processes, or setting up deployment automation. |
| `doc-coauthoring` | `skills/doc-coauthoring/SKILL.md` | Guide users through a structured workflow for co-authoring documentation. Use when the user wants to write documentation, proposals, technical specs, decision docs, or similar structured content. |
| `docx` | `skills/docx/SKILL.md` | Create, read, edit, or manipulate Word documents (.docx files). Use whenever the user mentions 'Word doc', '.docx', or requests professional documents with formatting. |
| `engineering-workflow-orchestrator` | `skills/engineering-workflow-orchestrator/SKILL.md` | Coordinate the complete software development lifecycle by selecting and sequencing the appropriate sub-skills. Use as the entry point for complex, multi-phase engineering tasks. |
| `frontend-application-development` | `skills/frontend-application-development/SKILL.md` | Build production-quality, fast, accessible, and maintainable web interfaces. Use when developing frontend features, UI components, React/Vue/Angular applications. |
| `frontend-design` | `skills/frontend-design/SKILL.md` | Create distinctive, production-grade frontend interfaces with high design quality. Use when building web components, pages, artifacts, or applications. |
| `incident-response-and-debugging` | `skills/incident-response-and-debugging/SKILL.md` | Diagnose and resolve production incidents systematically and safely. Use when investigating bugs, production outages, performance regressions, or any system failure. |
| `infrastructure-as-code` | `skills/infrastructure-as-code/SKILL.md` | Manage all infrastructure as versioned, tested, and repeatable code. Use when provisioning cloud resources or managing infrastructure with Terraform, Pulumi, CloudFormation, or similar tools. |
| `internal-comms` | `skills/internal-comms/SKILL.md` | Write internal communications. Use when asked to write status reports, leadership updates, company newsletters, FAQs, incident reports, project updates, or similar communications. |
| `mcp-builder` | `skills/mcp-builder/SKILL.md` | Create high-quality MCP (Model Context Protocol) servers. Use when building MCP servers to integrate external APIs or services, whether in Python or Node/TypeScript. |
| `observability-and-monitoring` | `skills/observability-and-monitoring/SKILL.md` | Instrument systems for full observability through structured logging, metrics, and distributed tracing. Use when setting up monitoring dashboards, alerting rules, or log aggregation. |
| `pdf` | `skills/pdf/SKILL.md` | Work with PDF files — reading, combining, splitting, creating, filling forms, encrypting, or performing OCR. Use whenever the user mentions a .pdf file or asks to produce one. |
| `performance-optimization` | `skills/performance-optimization/SKILL.md` | Optimize system performance. Use when diagnosing latency, throughput bottlenecks, memory problems, or profiling and improving code or infrastructure performance. |
| `pptx` | `skills/pptx/SKILL.md` | Create, read, edit, or manipulate PowerPoint files (.pptx). Use whenever the user mentions 'deck', 'slides', 'presentation', or references a .pptx filename. |
| `project-documentation` | `skills/project-documentation/SKILL.md` | Produce accurate, immediately useful technical documentation. Use when writing READMEs, API reference docs, ADRs, runbooks, onboarding guides, or any technical documentation. |
| `security-hardening` | `skills/security-hardening/SKILL.md` | Build systems that are secure by default. Use when auditing for vulnerabilities, implementing security controls, hardening configurations, addressing OWASP concerns, or performing threat modeling. |
| `skill-creator` | `skills/skill-creator/SKILL.md` | Create new skills, modify existing skills, and measure skill performance. Use when creating a skill from scratch, updating an existing skill, or running evals to test a skill. |
| `slack-gif-creator` | `skills/slack-gif-creator/SKILL.md` | Create animated GIFs optimized for Slack. Use when the user requests an animated GIF for Slack. |
| `testing-and-validation` | `skills/testing-and-validation/SKILL.md` | Build test suites that give the team confidence to ship quickly and safely. Use when writing unit, integration, or end-to-end tests, or increasing test coverage. |
| `theme-factory` | `skills/theme-factory/SKILL.md` | Style artifacts with a theme (slides, docs, reports, HTML pages). Use when applying or generating visual themes for any artifact. |
| `algorithmic-art` | `skills/algorithmic-art/SKILL.md` | Create algorithmic art using p5.js with seeded randomness and interactive parameter exploration. Use when the user requests generative art, flow fields, or particle systems. |
| `video` | `skills/video/SKILL.md` | Work with video files (.mp4, .mov, .avi, etc.) — extracting frames, analyzing content, extracting audio, transcribing speech, detecting scenes, or converting formats. |
| `web-artifacts-builder` | `skills/web-artifacts-builder/SKILL.md` | Create elaborate, multi-component HTML artifacts using React, Tailwind CSS, and shadcn/ui. Use for complex artifacts requiring state management, routing, or shadcn/ui components. |
| `webapp-testing` | `skills/webapp-testing/SKILL.md` | Interact with and test local web applications using Playwright. Use when verifying frontend functionality, debugging UI behavior, or capturing browser screenshots. |
| `xlsx` | `skills/xlsx/SKILL.md` | Work with spreadsheet files (.xlsx, .xlsm, .csv, .tsv) — reading, editing, creating, or converting. Use whenever the user wants to open, read, edit, or produce a spreadsheet. |

### Workflow

For any non-trivial engineering task, start with `engineering-workflow-orchestrator`, which will guide you through the right sequence of skills. For smaller or more targeted tasks, select the most relevant skills directly from the table above.

> **Important**: The skills in this workspace represent the team's accumulated knowledge and best practices. Always prefer skill-guided behavior over ad-hoc approaches.
