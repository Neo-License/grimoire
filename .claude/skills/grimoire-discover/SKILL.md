---
name: grimoire-discover
description: Generate per-area conventions files and data schema using codebase-memory-mcp. Requires MCP to be installed.
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: kiwi-data
  version: "0.1"
---

# grimoire-discover

Generate a structured project map in `.grimoire/docs/` from a codebase snapshot. This map helps LLMs understand the codebase layout, find reusable code, and follow existing patterns — preventing duplicate code and misplaced files.

## Triggers
- User wants to document or map the codebase structure
- User asks about coding standards, patterns, or conventions
- User wants to prevent duplicate code or find existing utilities
- Loose match: "discover", "map", "standards", "conventions", "DRY", "utilities", "codebase layout"

## Routing
- Want to document existing behavior as Gherkin features → `grimoire-audit`
- Want to find undocumented features and decisions → `grimoire-audit` (run discover first, then audit)
- Want to draft new functionality → `grimoire-draft`

## Prerequisites

**MCP required:** `codebase-memory-mcp` must be installed and indexed. If it is not available, stop immediately and tell the user:
- "codebase-memory-mcp is required for grimoire-discover."
- "Install it from: [MCP installation instructions]"
- "After installing, tell your agent to index this project, then re-run /grimoire:discover."

Do not proceed without MCP. Do not fall back to reading source files for symbol discovery.

## What It Produces

`.grimoire/docs/conventions/` with:
- **Per-area conventions files** — one markdown file per area (e.g., `api.md`, `models.md`), each covering:
  - File placement rules (where new code of this type goes)
  - Naming conventions (with examples)
  - Pattern guidance (what exemplar files to follow)
  - A "Last updated" date
- **NOT included**: reusable utility tables, full API inventories, call graphs — those are answered on demand by MCP queries

`.grimoire/docs/data/schema.yml` (if a data layer exists)
`.grimoire/docs/context.yml` (deployment and infrastructure context)
`.grimoire/docs/components.md` (if a UI component library is present)

## Workflow

### 1. Archive Legacy Area Docs
Before generating anything, check whether `.grimoire/docs/` contains legacy area doc files (any `.md` files directly in `.grimoire/docs/`, NOT in the `conventions/` subdirectory, NOT `context.yml`, NOT `components.md`).

If legacy docs exist:
1. Create `.grimoire/archive/docs/YYYY-MM-DD/` (today's date)
2. Move each legacy `.md` file there (including `index.yml` if present)
3. Print a note listing each archived file: "Archived legacy doc: .grimoire/docs/api.md → .grimoire/archive/docs/2026-05-21/api.md"

Then proceed. If no legacy docs exist, skip silently.

### 2. Determine Scope
Ask the user what to document:
- **Full scan** — document all areas from the snapshot (default for first run)
- **Area scan** — document specific directories (e.g., "just the API layer")
- **Gap fill** — only document areas not yet covered in `.grimoire/docs/conventions/`

### 3. Analyze Each Area
For each area identified by MCP's `get_architecture` output:

**From `codebase-memory-mcp` graph (required):**
- All symbols in the area: functions, classes, types, constants with signatures
- Call graph: what calls what, both inbound and outbound
- Dead code: functions with zero callers
- Cross-service HTTP links: REST routes and their callers

### 4. Generate Conventions Files
For each significant area, create `.grimoire/docs/conventions/<area>.md`.

**Conventions file format:**
```markdown
# <Area Name> Conventions
> Last updated: YYYY-MM-DD

## File Placement
- New <type> → `<path/to/directory/>`
- New <type> → `<path/to/other/>`

## Naming
- <naming convention with example from codebase>

## Patterns
- Follow `<path/to/exemplar/file.ts>` for <what it exemplifies>
- <structural pattern with example file reference>
```

**Rules:**
- Conventions files document PLACEMENT, NAMING, and PATTERNS only
- Do NOT include reusable utility tables (MCP answers those on demand)
- Do NOT include full file lists or API inventories
- Reference exemplar files by path; do NOT copy code
- One file per logical area; keep files under 50 lines

### 5. Generate Data Schema

Scan the codebase for data models, ORM definitions, migration files, and schema declarations. Produce `.grimoire/docs/data/schema.yml` documenting the current data layer.

**Where to look:**
- ORM models: Django `models.py`, SQLAlchemy models, Prisma `schema.prisma`, TypeORM entities, Mongoose schemas
- Migrations: `migrations/`, `alembic/versions/`, `prisma/migrations/`
- Raw SQL: `*.sql` files, schema definitions
- NoSQL: Mongoose schemas, DynamoDB table definitions, Firestore rules
- API schemas: GraphQL `.graphql` files, protobuf `.proto` files, JSON Schema
- External APIs: OpenAPI/Swagger specs, Postman collections, API client wrappers, SDK config files
- Message formats: Avro `.avsc`, protobuf `.proto`, JSON Schema for events/messages

**Schema format:** See `../references/schema-format.md` for the full YAML format with examples covering tables, nested objects, relationships, and external APIs.

**Rules:**
- Document what exists in the code, not what the database actually contains
- Use `source:` to point back to the ORM model or migration file — the schema.yml is a summary, the code is the truth
- Use `type: table` for SQL, `type: collection` for Mongo/document stores, `type: document` for nested sub-documents
- Use `type: external_api` for APIs you consume or produce but don't own the schema for
- Nested `fields` for embedded objects/arrays (common in document DBs and JSON columns)
- Include `note:` only when the field name isn't self-explanatory
- Include `relationships` when the ORM defines them explicitly
- For external APIs: `schema_ref` is the most important field — point to the OpenAPI spec, Swagger URL, API docs page, or local spec file so the LLM (and humans) know where to get the full contract
- For external APIs: `client` points to where the codebase calls the API — this is where changes happen when the API changes
- Don't duplicate entire OpenAPI specs into schema.yml — summarize the endpoints you actually use with key fields, and point to the full spec via `schema_ref`
- If the project has no data layer, skip this step entirely

If `.grimoire/docs/data/` already exists, update it rather than regenerating. Diff against existing schema.yml to flag new models or removed fields.

### 5.5 Component Inventory (optional)

Scan the codebase for an existing UI component library, then produce `.grimoire/docs/components.md` documenting reusable components. This inventory lets `grimoire-design` reuse what exists instead of generating duplicate components.

**Detection — component library:**

| Signal | What it tells you |
|--------|------------------|
| `components.json` | shadcn/ui — components live under the configured `aliases.components` path (typically `components/ui/`) |
| `tailwind.config.{js,ts}` | Tailwind project — utility-first; components are project-local |
| `package.json` deps: `@mui/material` | Material UI — components imported from `@mui/material/*` |
| `package.json` deps: `@chakra-ui/react` | Chakra UI — components imported from `@chakra-ui/react` |
| `package.json` deps: `@mantine/core` (or any `mantine` package) | Mantine |
| `package.json` deps: `@radix-ui/*` | Radix primitives — usually wrapped by shadcn or project components |

**Detection — Storybook:**

| Signal | What it tells you |
|--------|------------------|
| `.storybook/main.{ts,js}` | Storybook configured — stories define canonical component variants |
| `*.stories.{ts,tsx,jsx,js}` | Story files — each story is a documented component variant |

**Skip condition:** If no library signal and no story files are found, emit a single-line note ("No UI component signals detected — skipping component inventory.") and continue to §6. Do not create `components.md`.

**Workflow:**
1. Detect the library (or libraries) using the signals above
2. Locate component source files — for shadcn, walk `components/ui/`; for project-local components, look under `src/components/`, `app/components/`, or wherever the convention places them
3. For each component file, extract: name, file path, exported variants (e.g., `variant="primary|secondary"`), and notable props (especially required ones)
4. If Storybook is present, walk `*.stories.*` files to harvest the canonical variant list per component — stories are the source of truth for which variants exist
5. Write `.grimoire/docs/components.md` listing each component with file path, variants, and key props

**`components.md` format:**

```markdown
# Component Inventory
> Last updated: YYYY-MM-DD
> Library: <shadcn | MUI | Chakra | Mantine | project-local | mixed>

## Components

| Component | Location | Variants | Key Props | Notes |
|-----------|----------|----------|-----------|-------|
| `Button` | `components/ui/button.tsx` | `default`, `destructive`, `outline`, `ghost`, `link` | `variant`, `size`, `asChild` | Wraps Radix Slot when `asChild` |
| `Dialog` | `components/ui/dialog.tsx` | — | `open`, `onOpenChange` | Compound: `Dialog`, `DialogTrigger`, `DialogContent` |

## Stories
<Only if Storybook detected. List story files and the variants they document.>

| Story File | Component | Documented Variants |
|------------|-----------|---------------------|
| `Button.stories.tsx` | `Button` | Primary, Secondary, Destructive, Loading |
```

**Rules:**
- Document what exists in code, not what should exist. If the project has both shadcn and ad-hoc components, list both and note the inconsistency.
- Point to source files; do not duplicate component code into the doc.
- Variants come from prop unions in the type signature OR from the canonical Storybook stories — prefer Storybook when present.
- Only list components meant for reuse. Skip one-off page-level components (e.g., `LoginPage`) unless they're imported elsewhere.
- If `.grimoire/docs/components.md` already exists, update it — diff against existing entries to flag new or removed components.

### 6. Generate Project Context

Scan the codebase for deployment and infrastructure artifacts, then populate `.grimoire/docs/context.yml`. This file captures the project's ecosystem — how it's deployed, what services it talks to, and what infrastructure it depends on. If `context.yml` doesn't exist, copy it from the template first (`grimoire init` creates it, but this handles projects initialized before this feature).

**Where to look:**

| Artifact | What it tells you |
|----------|------------------|
| `Dockerfile`, `docker-compose.yml` | Containerized deployment; compose reveals linked services, databases, caches |
| `k8s/`, `kubernetes/`, `Chart.yaml`, `helmfile.yaml` | Kubernetes deployment; manifests reveal services, ingresses, config maps |
| `*.tf`, `terraform/`, `cdk.json`, `serverless.yml` | Infrastructure-as-code; reveals cloud provider, services, and architecture |
| `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/` | CI/CD platform and deploy triggers |
| `Procfile`, `app.json`, `vercel.json`, `netlify.toml` | PaaS deployment target |
| `fly.toml`, `render.yaml`, `railway.json` | PaaS deployment target |
| `.env.example`, `.env.template` | Environment variables reveal infrastructure dependencies (DB hosts, cache URLs, API keys) |
| `docker-compose.yml` services | Related services, databases, caches, queues running locally |
| API client wrappers, SDK config | Internal service dependencies |

**Workflow:**
1. Scan for the artifacts above — note what exists
2. Read `docker-compose.yml` (if present) — it's the richest source of service and infrastructure dependencies
3. Read `.env.example` (if present) — environment variables reveal what the project connects to
4. Read CI/CD config files — identify the platform, key workflows, and deploy triggers
5. Read IaC files (Terraform, CDK, etc.) — identify cloud provider and provisioned resources
6. Populate `context.yml` with what you found — fill in real values, remove unused commented sections
7. Present findings to the user for confirmation — they'll know about services and infrastructure that aren't discoverable from code alone (e.g., a shared auth service, a data warehouse they push to)

**Rules:**
- Only populate sections where you found evidence. Leave sections empty (with comments) rather than guessing.
- Use environment variable references (`${DATABASE_HOST}`) for hostnames and credentials — never hardcode real values.
- The `services` section is for **internal/sibling services** your org owns. Third-party APIs (Stripe, Twilio, etc.) belong in `schema.yml` under `external_api`.
- If `context.yml` already exists and has content, update it rather than overwriting — the user may have manually added entries.
- Ask the user about anything you can't determine from code: "I see a Redis connection in docker-compose but I'm not sure if it's just cache or also used for sessions — which is it?"

### 7. Present Summary
After generating, show the user:
- How many conventions files were generated
- Which areas were documented
- Any areas that seem under-organized or have pattern inconsistencies
- Suggest which conventions files are most critical for the plan skill to read

## Integration with Other Skills

- The **plan** skill reads `.grimoire/docs/conventions/<area>.md` for placement/naming guidance and queries MCP directly for symbol/utility lookup
- The **audit** skill can trigger a discover pass during onboarding; it also uses conventions files for drift detection
- The **apply** skill's context blocks reference conventions files, not area docs
- Run `/grimoire:audit` with scope "conventions" to detect drift after the codebase evolves significantly

## Freshness Tracking

Every conventions file must include a `> Last updated:` date at the top. This lets other skills judge whether the docs are trustworthy or stale.

**In each conventions file**, include a last-updated line:
```markdown
# API Conventions
> Last updated: 2026-04-05
```

## Important
- **MCP is required and is the only discovery path.** No fallback to file reads for symbol discovery.
- **Conventions files are small by design.** If a conventions file exceeds ~50 lines, you're putting too much in it — move symbol/API detail to MCP queries.
- **Archive, don't delete.** Legacy area docs go to `.grimoire/archive/docs/YYYY-MM-DD/` on first run.
- **Document what IS, not what should be.** This is a map of the actual codebase, not aspirational standards. If the code is inconsistent, note it — don't paper over it.
- **Point, don't copy.** Reference files and line numbers as exemplars. Don't duplicate code into the docs — it goes stale.
- **Keep docs lean.** Each conventions file should be scannable in 30 seconds.

## Done
When conventions files, schema, context, and component inventory are generated, the workflow is complete. Suggest `grimoire-audit` to document existing features and decisions as Gherkin specs and ADRs.
