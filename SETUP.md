# Setup Guide — Claude Sub-Agents for Legacy Migration

This document covers how to install, configure, and run the Claude sub-agents that power the Legacy Migration POC. The system uses **Claude Code CLI** with custom agent definitions to generate PRDs and migrate legacy Java/Swing forms to .NET 9 + React 18 + PostgreSQL.

## System Architecture

![Sub-Agent Architecture Flow](SubAgent_Flow.png)

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Project Structure](#2-project-structure)
3. [Installation](#3-installation)
4. [Agent Overview](#4-agent-overview)
5. [Running the Agents (CLI Commands)](#5-running-the-agents-cli-commands)
6. [Adding a New Form for Migration](#6-adding-a-new-form-for-migration)
7. [Agent Memory](#7-agent-memory)
8. [Output Locations](#8-output-locations)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Prerequisites

| Requirement           | Version | Notes                                     |
| --------------------- | ------- | ----------------------------------------- |
| **Claude Code CLI**   | Latest  | The `claude` CLI tool from Anthropic      |
| **Node.js**           | 18+     | Required by Claude Code CLI               |
| **macOS / Linux**     | —       | Windows via WSL2 is also supported        |
| **Anthropic API Key** | —       | Or an active Claude Pro/Team subscription |

### Install Claude Code CLI

```bash
# Install via npm (recommended)
npm install -g @anthropic-ai/claude-code

# Verify installation
claude --version
```

### Authenticate

```bash
# Login with your Anthropic account (opens browser)
claude login

# Or set API key directly
export ANTHROPIC_API_KEY="sk-ant-..."
```

---

## 2. Project Structure

```
v0_sub_agents/
├── .claude/
│   ├── agents/                              # Agent definitions
│   │   ├── legacy-form-prd-generator.md     # PRD generation agent
│   │   └── legacy-migration-orchestrator.md # Code migration agent
│   └── agent-memory-local/                  # Persistent agent memory
│       └── legacy-migration-orchestrator/
│           └── MEMORY.md
├── metadata/                                # Input data for agents
│   ├── DB_PRD/
│   │   └── Database_DOC.md                  # Database schema documentation
│   ├── EXPORT_CODEBASE_PRD/
│   │   ├── BE/
│   │   │   └── dotnet_backend_conversion_prompt.txt
│   │   └── FE/
│   │       └── react_frontend_conversion_prompt.txt
│   ├── FORMS/
│   │   ├── IS13/                            # Per-form metadata
│   │   │   ├── FORM_DOCS/
│   │   │   ├── FORM_FILE_DEPENDENCIES/
│   │   │   │   └── is13_dependencies.txt
│   │   │   └── UI_SCREENSHOTS/
│   │   └── LE11/
│   │       ├── FORM_DOCS/
│   │       ├── FORM_FILE_DEPENDENCIES/
│   │       │   └── le11_dependencies.txt
│   │       └── UI_SCREENSHOTS/
│   └── LEGACY_CODEBASE/
│       └── oases-master/                    # Full legacy Java codebase
├── migrations/                              # Generated migration code output
│   └── LE11/
│       ├── DotNet/                          # .NET 9 backend
│       └── React/                           # React 18 frontend
├── output/
│   └── PRD/                                 # Generated PRD documents
│       └── IS13_PRD.md
└── SETUP.md                                 # ← You are here
```

---

## 3. Installation

### Clone / Copy the Project

```bash
# If hosted in a repo
git clone <your-repo-url> v0_sub_agents
cd v0_sub_agents
```

### Verify Agent Definitions Exist

```bash
# Both agent files must be present under .claude/agents/
ls -la .claude/agents/
# Expected output:
#   legacy-form-prd-generator.md
#   legacy-migration-orchestrator.md
```

### Verify Metadata is in Place

```bash
# Check that the legacy codebase is present
ls metadata/LEGACY_CODEBASE/oases-master/src/main/java/ | head -5

# Check that at least one form has its metadata
ls metadata/FORMS/
# Expected: IS13  LE11  (or whichever forms you have)

# Check form dependencies exist
cat metadata/FORMS/IS13/FORM_FILE_DEPENDENCIES/is13_dependencies.txt | head -5

# Check database docs
wc -l metadata/DB_PRD/Database_DOC.md
# Expected: ~21,000+ lines

# Check conversion prompts
ls metadata/EXPORT_CODEBASE_PRD/BE/
ls metadata/EXPORT_CODEBASE_PRD/FE/
```

### Create Output Directories (Optional)

The agents use the `Write` tool which auto-creates directories, but you can pre-create them:

```bash
mkdir -p output/PRD
mkdir -p migrations
```

---

## 4. Agent Overview

### Agent 1: `legacy-form-prd-generator`

| Property      | Value                                                            |
| ------------- | ---------------------------------------------------------------- |
| **Purpose**   | Generate a comprehensive PRD for a legacy form                   |
| **Model**     | `sonnet`                                                         |
| **Max Turns** | 100                                                              |
| **Memory**    | Local (`.claude/agent-memory-local/legacy-form-prd-generator/`)  |
| **Input**     | Form dependencies, Java source code, UI screenshots, DB docs     |
| **Output**    | `/output/PRD/{FORM_ID}_PRD.md`                                   |
| **Behavior**  | Fully autonomous — completes in one run without asking questions |

### Agent 2: `legacy-migration-orchestrator`

| Property     | Value                                                                  |
| ------------ | ---------------------------------------------------------------------- |
| **Purpose**  | Generate complete .NET 9 backend + React 18 frontend code              |
| **Model**    | `sonnet`                                                               |
| **Memory**   | Local (`.claude/agent-memory-local/legacy-migration-orchestrator/`)    |
| **Input**    | Conversion prompts, form dependencies, Java source code                |
| **Output**   | `/migrations/{FORM_ID}/Backend/` and `/migrations/{FORM_ID}/Frontend/` |
| **Behavior** | Generates 20–50+ actual code files per migration                       |

---

## 5. Running the Agents (CLI Commands)

All commands should be run from the project root (`v0_sub_agents/`).

### 5.1 Generate a PRD

Use the **legacy-form-prd-generator** agent to create a Product Requirements Document for a specific form.

```bash
# Generate PRD for form IS13
claude --agent legacy-form-prd-generator "generate prd is13"

# Generate PRD for form LE11
claude --agent legacy-form-prd-generator "generate prd le11"

# Generic pattern for any form
claude --agent legacy-form-prd-generator "generate prd <FORM_ID>"
```

**What happens:**

1. Agent reads `metadata/FORMS/{FORM_ID}/FORM_FILE_DEPENDENCIES/{form_id}_dependencies.txt`
2. Agent reads all referenced Java source files from the legacy codebase
3. Agent analyzes UI screenshots in `metadata/FORMS/{FORM_ID}/UI_SCREENSHOTS/`
4. Agent cross-references `metadata/DB_PRD/Database_DOC.md`
5. Agent writes the complete PRD to `output/PRD/{FORM_ID}_PRD.md`

**Example output:**

```
✅ PRD Generation Complete | Total Tokens: ~150,000 | Output: /output/PRD/IS13_PRD.md
```

### 5.2 Migrate a Form (Generate Code)

Use the **legacy-migration-orchestrator** agent to generate the full .NET + React codebase.

```bash
# Migrate form LE11
claude --agent legacy-migration-orchestrator "migrate le11"

# Migrate form IS13
claude --agent legacy-migration-orchestrator "migrate is13"

# Generic pattern for any form
claude --agent legacy-migration-orchestrator "migrate <FORM_ID>"
```

**What happens:**

1. Agent reads conversion prompts from `metadata/EXPORT_CODEBASE_PRD/BE/` and `FE/`
2. Agent reads form dependencies and all referenced Java source files
3. Agent extracts every line of business logic from Java code
4. Agent generates complete .NET 9 backend code under `migrations/{FORM_ID}/Backend/`
5. Agent generates complete React 18 frontend code under `migrations/{FORM_ID}/Frontend/`

**Expected output files (~20–50+ files):**

```
migrations/{FORM_ID}/
├── Backend/
│   ├── {Project}.API/Controllers/*.cs
│   ├── {Project}.API/Program.cs
│   ├── {Project}.API/appsettings.json
│   ├── {Project}.Business/Services/**/*.cs
│   ├── {Project}.Business/DTOs/**/*.cs
│   ├── {Project}.Business/Validators/*.cs
│   ├── {Project}.Business/Mappers/*.cs
│   ├── {Project}.Data/Entities/*.cs
│   ├── {Project}.Data/Configurations/*.cs
│   ├── {Project}.Data/Context/*.cs
│   ├── {Project}.Data/Repositories/**/*.cs
│   └── {Project}.Common/**/*.cs
└── Frontend/
    └── src/
        ├── types/*.ts
        ├── schemas/*.schema.ts
        ├── hooks/api/*.ts
        ├── components/**/*.tsx
        └── lib/api.ts
```

### 5.3 Interactive Mode

You can also start an interactive Claude session with a specific agent:

```bash
# Start interactive session with the PRD generator
claude --agent legacy-form-prd-generator

# Start interactive session with the migration orchestrator
claude --agent legacy-migration-orchestrator
```

In interactive mode, type your request at the prompt:

```
> generate prd is13
> migrate le11
> how does the migration process work?
```

### 5.4 Useful CLI Flags

```bash
# Run with verbose output to see agent thinking
claude --agent legacy-form-prd-generator --verbose "generate prd is13"

# Resume a previous conversation (if the agent was interrupted)
claude --resume

# Print token usage after completion
claude --agent legacy-migration-orchestrator --usage "migrate le11"

# Use a specific model override (e.g., opus for complex forms)
claude --agent legacy-form-prd-generator --model claude-sonnet-4-20250514 "generate prd is13"

# Dry run — see what the agent would do without writing files
claude --agent legacy-migration-orchestrator --dry-run "migrate is13"
```

### 5.5 Full Workflow Example (PRD then Migrate)

```bash
# Step 1: Generate the PRD first (analysis + documentation)
claude --agent legacy-form-prd-generator "generate prd is13"

# Step 2: Review the generated PRD
cat output/PRD/IS13_PRD.md | head -100

# Step 3: Run the migration (code generation)
claude --agent legacy-migration-orchestrator "migrate is13"

# Step 4: Inspect generated code
find migrations/IS13 -type f | head -30

# Step 5: Check the backend structure
ls -R migrations/IS13/Backend/

# Step 6: Check the frontend structure
ls -R migrations/IS13/Frontend/src/
```

---

## 6. Adding a New Form for Migration

To add a new form (e.g., `AB99`) for PRD generation or migration:

### Step 1: Create the Form Metadata Directory

```bash
FORM_ID="AB99"

mkdir -p "metadata/FORMS/${FORM_ID}/FORM_DOCS"
mkdir -p "metadata/FORMS/${FORM_ID}/FORM_FILE_DEPENDENCIES"
mkdir -p "metadata/FORMS/${FORM_ID}/UI_SCREENSHOTS"
```

### Step 2: Create the Dependencies File

List all Java source files that the form depends on. One file path per line, relative to `metadata/LEGACY_CODEBASE/oases-master/`.

### Step 3: Add UI Screenshots

Copy screenshots of the legacy form's UI into the screenshots folder:

```bash
cp /path/to/screenshots/*.png "metadata/FORMS/${FORM_ID}/UI_SCREENSHOTS/"
```

### Step 4: Run the Agents

```bash
# Generate PRD
claude --agent legacy-form-prd-generator "generate prd ${FORM_ID}"

# Generate migration code
claude --agent legacy-migration-orchestrator "migrate ${FORM_ID}"
```

---

## 7. Agent Memory

Both agents support **local persistent memory** that accumulates knowledge across sessions.

### Memory Locations

```
.claude/agent-memory-local/
├── legacy-form-prd-generator/
│   └── MEMORY.md          # PRD agent's learned patterns
└── legacy-migration-orchestrator/
    └── MEMORY.md           # Migration agent's learned patterns
```

### How Memory Works

- Agents automatically read `MEMORY.md` at the start of each session
- As agents discover patterns (component mappings, business logic conventions, etc.), they write notes to memory
- Memory persists across conversations — each run builds on prior knowledge
- `MEMORY.md` is truncated at 200 lines in the system prompt; detailed notes go in separate topic files

### Viewing Agent Memory

```bash
# View the migration orchestrator's accumulated knowledge
cat .claude/agent-memory-local/legacy-migration-orchestrator/MEMORY.md

# View the PRD generator's memory (may be empty initially)
cat .claude/agent-memory-local/legacy-form-prd-generator/MEMORY.md
```

### Resetting Agent Memory

```bash
# Reset a specific agent's memory
echo "" > .claude/agent-memory-local/legacy-migration-orchestrator/MEMORY.md

# Or remove the entire memory directory
rm -rf .claude/agent-memory-local/legacy-migration-orchestrator/
```

---

## 8. Output Locations

| Agent                             | Output Path                      | File Types             |
| --------------------------------- | -------------------------------- | ---------------------- |
| PRD Generator                     | `output/PRD/{FORM_ID}_PRD.md`    | `.md`                  |
| Migration Orchestrator (Backend)  | `migrations/{FORM_ID}/Backend/`  | `.cs`, `.json`         |
| Migration Orchestrator (Frontend) | `migrations/{FORM_ID}/Frontend/` | `.tsx`, `.ts`, `.json` |

### Inspecting Outputs

```bash
# List all generated PRDs
ls -la output/PRD/

# Count files generated for a migration
find migrations/LE11 -type f | wc -l

# List all backend C# files
find migrations/LE11 -name "*.cs" -type f

# List all frontend TypeScript/React files
find migrations/LE11 -name "*.tsx" -o -name "*.ts" | grep -v node_modules

# View a generated controller
cat migrations/LE11/DotNet/Controllers/FleetChapterController.cs
```

---

## 9. Troubleshooting

### Agent not found

```
Error: Agent "legacy-form-prd-generator" not found
```

**Fix:** Ensure you are running the command from the project root where `.claude/agents/` exists:

```bash
cd /path/to/v0_sub_agents
ls .claude/agents/
```

### Form metadata not found

```
Error: File not found: metadata/FORMS/XY99/FORM_FILE_DEPENDENCIES/xy99_dependencies.txt
```

**Fix:** Create the form metadata structure. See [Section 6](#6-adding-a-new-form-for-migration).

### Agent runs out of context / stops mid-generation

The migration orchestrator reads large Java files and generates many output files. If it hits the context limit:

```bash
# Resume the previous conversation
claude --resume

# Or re-run with a prompt that focuses on remaining work
claude --agent legacy-migration-orchestrator "continue migrating is13 — backend is done, generate frontend only"
```

### Token usage is very high

PRD generation for complex forms can consume 100,000–200,000+ tokens. To monitor:

```bash
claude --agent legacy-form-prd-generator --usage "generate prd is13"
```

### Java source files not being read

Ensure the dependency file lists paths relative to the `oases-master` root:

```bash
# Correct format in dependencies file:
src/main/java/options/is13.java

# NOT:
metadata/LEGACY_CODEBASE/oases-master/src/main/java/options/is13.java
/absolute/path/to/is13.java
```

### Claude CLI authentication issues

```bash
# Re-authenticate
claude login

# Or verify your API key
echo $ANTHROPIC_API_KEY

# Test with a simple command
claude "hello"
```

---

## Quick Reference Card

```bash
# ─── PRD Generation ───────────────────────────────────
claude --agent legacy-form-prd-generator "generate prd is13"
claude --agent legacy-form-prd-generator "generate prd le11"
claude --agent legacy-form-prd-generator "generate prd <FORM_ID>"

# ─── Code Migration ───────────────────────────────────
claude --agent legacy-migration-orchestrator "migrate le11"
claude --agent legacy-migration-orchestrator "migrate is13"
claude --agent legacy-migration-orchestrator "migrate <FORM_ID>"

# ─── Interactive Sessions ─────────────────────────────
claude --agent legacy-form-prd-generator
claude --agent legacy-migration-orchestrator

# ─── Inspect Outputs ──────────────────────────────────
cat output/PRD/IS13_PRD.md
find migrations/IS13 -type f
ls -R migrations/IS13/Backend/
ls -R migrations/IS13/Frontend/src/

# ─── Agent Memory ─────────────────────────────────────
cat .claude/agent-memory-local/legacy-migration-orchestrator/MEMORY.md
cat .claude/agent-memory-local/legacy-form-prd-generator/MEMORY.md

# ─── Add New Form ─────────────────────────────────────
mkdir -p metadata/FORMS/{FORM_ID}/{FORM_DOCS,FORM_FILE_DEPENDENCIES,UI_SCREENSHOTS}
# Then populate dependencies file and screenshots
```
