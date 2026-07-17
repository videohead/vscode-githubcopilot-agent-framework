# Multi-Agent Systems in VS Code Copilot

A comprehensive guide to creating and working with multi-agent systems in Visual Studio Code using GitHub Copilot.

---

## Overview

As of VS Code v1.109 (January 2026), Visual Studio Code has become a **multi-agent command center**. You can now run GitHub Copilot, Claude, and OpenAI Codex agents side by side — locally or in the cloud — all from a single interface.

### What makes it "multi-agent"?

Instead of relying on a single AI assistant, you can now:

- Run multiple agents simultaneously (Copilot, Claude, Codex)
- Create **custom agents** with specialized roles (planner, reviewer, implementer)
- Delegate subtasks to **subagents** that run in parallel with isolated context
- **Hand off** tasks between agents to leverage each one's strengths
- Manage everything from a unified **Agent Sessions** view

---

## Prerequisites

1. **VS Code** v1.109 or later
2. **GitHub Copilot subscription** — Pro, Pro+, Business, or Enterprise
3. **Agent mode enabled** — set `chat.agent.enabled` to `true` in VS Code settings

## Agent types in VS Code

### Local agents

Run interactively in VS Code with full access to your workspace, tools, and models. Best for tasks where you want immediate feedback and results.

### Copilot CLI agents

Run autonomously in the background on your machine, optionally using Git worktrees for isolation. Ideal for delegating tasks that don't require immediate interaction.

### Cloud agents

Run on remote infrastructure and integrate with GitHub pull requests for team collaboration. Best for longer-running tasks or collaborative reviews.

### Third-party agents

Connect external AI providers like Anthropic (Claude) and OpenAI (Codex), with options for running locally or in the cloud.

---

## Custom agents

Custom agents are defined as `.agent.md` files. They allow you to tailor Copilot's expertise for specific tasks.

### File locations

| Location                         | Scope                                  |
|----------------------------------|----------------------------------------|
| `.github/agents/`               | Workspace — available in that repo     |
| User profile folder              | User — available across all workspaces |
| `.github-private` repo           | Organization/enterprise-wide           |


---
## Custom Models

Update each/any of the agent files with more advanced models. You can find and replace 'model: ['YOUR MODEL HERE (copilot)']'

To add a local model served by vLLM, NIM, or Ollama:
Click on the Models box in VSCode Copilot Chat
Click The Gear Icon next to Other Models
CLick Add Models +
Name Your new Custom Model Group

```markdown
[
	{
		"name": "Sparkles",
		"vendor": "customendpoint",
		"apiType": "chat-completions",
		"models": [
			{
				"id": "qwen3.6:35b-a3b-q4_K_M",
				"name": "qwen3.6:35b-a3b-q4_K_M",
				"url": "http://10.0.0.34:11434",
				"toolCalling": true,
				"vision": true,
				"maxInputTokens": 128000,
				"maxOutputTokens": 16000
			}
		]
	}
]
```

### Agent file structure

```markdown
---
name: Agent Name
description: What this agent does
tools: ['tool1', 'tool2']
model: ['YOUR MODEL HERE (copilot)']
---

# Agent instructions

Your detailed instructions go here in Markdown format.
```

### Key properties

| Property                  | Description                                                      |
|---------------------------|------------------------------------------------------------------|
| `name`                    | Display name in the agents dropdown                              |
| `description`             | What the agent does (helps with auto-invocation)                 |
| `tools`                   | List of tools the agent can use                                  |
| `model`                   | Single model or prioritized list (tries each in order)           |
| `agents`                  | List of agent names available as subagents (use `*` for all)     |
| `user-invocable`          | Whether agent appears in the dropdown (default: true)            |
| `disable-model-invocation`| Prevent auto-invocation as a subagent (default: false)           |
| `handoffs`                | Define transition buttons to other agents                        |

---

## Real-time example 1: Feature builder with subagents

This pattern coordinates a **Researcher** and **Implementer** subagent for a research-then-implement workflow.

### Coordinator — `.github/agents/feature-builder.agent.md`

```markdown
---
name: Feature Builder
description: Build features by researching first, then implementing
tools: ['agent']
agents: ['Researcher', 'Implementer']
model: ['YOUR MODEL HERE (copilot)']
---

You are a feature builder. For each task:

1. Use the Researcher agent to gather context and find relevant patterns in the codebase
2. Review the research findings
3. Use the Implementer agent to make the actual code changes based on research findings
4. Verify the implementation matches the research recommendations
```

### Researcher subagent — `.github/agents/researcher.agent.md`

```markdown
---
name: Researcher
description: Research codebase patterns and gather context
tools: ['codebase', 'fetch', 'usages', 'search']
user-invocable: false
model: ['YOUR MODEL HERE (copilot)']
---

You are a research specialist. Your job is to:

- Search the codebase for relevant patterns and conventions
- Identify existing implementations of similar features
- Note dependencies, testing patterns, and code style conventions
- Return a structured summary of findings

Do NOT make any code changes. Only read and analyze.
```

### Implementer subagent — `.github/agents/implementer.agent.md`

```markdown
---
name: Implementer
description: Implement code changes based on provided context
tools: ['editFiles', 'terminalLastCommand', 'run']
user-invocable: false
model: ['YOUR MODEL HERE (copilot)']
---

Implement changes based on the research findings provided to you.
Follow the patterns and conventions identified by the researcher.
Write clean, tested code that matches the existing codebase style.
```

### How to use it

Open Chat, select **Feature Builder** from the agents dropdown, then prompt:

```
Add a user notification system that sends email alerts
when their subscription is about to expire.
```

The Feature Builder will automatically:
1. Spin up the Researcher to analyze your codebase
2. Get back a summary of patterns and conventions
3. Spin up the Implementer with those findings
4. Return the completed implementation

---

## Real-time example 2: Plan → Implement → Review pipeline

This uses **handoffs** to create a guided sequential workflow.

### Planner — `.github/agents/planner.agent.md`

```markdown
---
name: Planner
description: Generate implementation plans for new features or refactoring
tools: ['fetch', 'githubRepo', 'search', 'usages']
model: ['YOUR MODEL HERE (copilot)']
handoffs:
  - label: Start Implementation
    agent: Builder
    prompt: Implement the plan outlined above.
    send: false
---

# Planning instructions

You are in planning mode. Your task is to generate a detailed
implementation plan. Do NOT make any code edits.

The plan should include:
- **Overview**: Brief description of the feature
- **Affected files**: List of files that need changes
- **Step-by-step changes**: Ordered list of modifications
- **Testing strategy**: How to verify the changes
- **Potential risks**: Edge cases and things to watch out for
```

### Builder — `.github/agents/builder.agent.md`

```markdown
---
name: Builder
description: Implement code changes based on a plan
tools: ['editFiles', 'terminalLastCommand', 'run', 'agent']
model: ['YOUR MODEL HERE (copilot)']
handoffs:
  - label: Review Changes
    agent: Code Reviewer
    prompt: Review all the changes made in this session.
    send: false
---

Implement the plan step by step. After each major change:
1. Run relevant tests
2. Fix any errors before moving to the next step
3. Commit logical units of work
```

### Code Reviewer — `.github/agents/code-reviewer.agent.md`

```markdown
---
name: Code Reviewer
description: Review code for quality, security, and performance
tools: ['read', 'search', 'codebase']
model: ['YOUR MODEL HERE (copliot)']
---

Review the code changes for:
- **Correctness**: Logic errors, edge cases, type issues
- **Security**: Vulnerabilities, injection risks, auth gaps
- **Performance**: N+1 queries, memory leaks, unnecessary allocations
- **Style**: Consistency with codebase conventions

Provide a structured review with severity levels (critical/warning/info).
```

### Workflow

1. Select **Planner** → enter your feature request
2. Review the plan → click **"Start Implementation"** handoff button
3. Builder implements the plan → click **"Review Changes"**
4. Code Reviewer provides feedback

Each step preserves the full conversation context.

---

## Real-time example 3: Parallel code review

Use subagents to run multiple review perspectives simultaneously.

### `.github/agents/thorough-reviewer.agent.md`

```markdown
---
name: Thorough Reviewer
description: Multi-perspective code review using parallel subagents
tools: ['agent', 'read', 'search']
model: ['YOUR MODEL HERE (copilot)']
agents: ['*']
---

You review code through multiple perspectives simultaneously.
When asked to review code, run these subagents **in parallel**:

1. **Correctness reviewer**: Check for logic errors, edge cases, type issues,
   and incorrect assumptions
2. **Security reviewer**: Scan for vulnerabilities, injection risks,
   authentication gaps, and data exposure
3. **Performance reviewer**: Identify N+1 queries, memory leaks,
   unnecessary allocations, and bottlenecks

After all subagents complete, consolidate their findings into a single
prioritized review summary organized by severity:
- 🔴 Critical — must fix before merge
- 🟡 Warning — should fix, potential issues
- 🔵 Info — suggestions for improvement
```

### Usage

```
Review the changes in the last 3 commits for the authentication module.
```

Three subagents spin up in parallel, each with isolated context, and the coordinator synthesizes results.

---

## Real-time example 4: Cross-agent workflow

Mix different agents across sessions to leverage each one's strengths.

### Step 1 — Plan with Claude Sonnet 4.6 (local)

Open Chat → select **Agent** → choose **Claude Sonnet 4.6**:

```
Analyze our codebase and create a plan for adding OAuth2 
authentication with Google and GitHub providers.
```

Claude excels at identifying missing context, edge cases, and asking clarifying questions.

### Step 2 — Implement with Copilot CLI (background)

Hand off the session to Copilot CLI:
- Select a different agent type from the session dropdown
- Copilot CLI creates a Git worktree and works autonomously
- You continue coding in your main workspace without conflicts

### Step 3 — Submit with cloud agent

Hand off to a cloud agent:
- Use the `/delegate` command in a Copilot CLI session
- The cloud agent creates a PR with the changes
- Team members review through GitHub's normal PR workflow

### Step 4 — Track everything

The **Agent Sessions** view in VS Code's sidebar shows all running sessions — local, background, and cloud — so you can monitor progress and switch between them.

---

## Skills system

Skills are folders of instructions and resources that Copilot loads on-demand when relevant. They work across Copilot, Claude, and Codex agents.

### Create a skill

Create a folder at `.github/skills/my-skill/` with a `SKILL.md` file:

```markdown
---
name: api-testing
description: Guide for testing REST APIs using Jest and Supertest.
  Use this when asked to create or run API tests.
---

# API Testing with Jest + Supertest

## Setup
- Install: `npm install --save-dev jest supertest`
- Create test files in `__tests__/api/`

## Conventions
- One test file per route group
- Use `describe` blocks for each endpoint
- Test success cases, validation errors, and auth failures
```

### Using skills

Type `/skills` in chat to see available skills, or just start a task — Copilot auto-loads relevant skills based on their descriptions.

---

## MCP (Model Context Protocol) servers

MCP servers extend agents with external tool access. Both Copilot and Claude support MCP integration.

### Adding MCP servers

Configure in VS Code settings or in your `.vscode/settings.json`:

```json
{
  "mcp.servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

### Using MCP tools in custom agents

```markdown
---
name: DB Explorer
description: Explore and query the database
tools: ['mcp_postgres_query', 'mcp_postgres_schema']
model: ['Claude Sonnet 4.6 (copilot)']
---

You help developers explore the database. Use the postgres tools
to inspect schema and run read-only queries.
```

---

## AGENTS.md — the cross-agent standard

Create an `AGENTS.md` file in your repo root to provide consistent instructions across all agents:

```markdown
# Agent Instructions

## Project overview
This is a Node.js/TypeScript REST API using Express, Prisma ORM,
and PostgreSQL.

## Coding conventions
- Use TypeScript strict mode
- Follow existing patterns in src/
- Write tests for all new endpoints
- Use Zod for request validation

## Testing
- Run tests: `npm test`
- Run specific: `npm test -- --grep "auth"`
- Tests must pass before committing

## Architecture
- Routes: src/routes/
- Services: src/services/
- Models: prisma/schema.prisma
- Tests: __tests__/
```

This file is respected by Copilot, Claude, and Codex agents alike.

---

## Quick reference: Agent file cheat sheet

```markdown
---
name: My Agent                              # Display name
description: What this agent does           # Auto-matching trigger
tools: ['editFiles', 'run', 'agent']        # Available tools
agents: ['SubAgent1', 'SubAgent2']          # Allowed subagents
model: ['Claude Sonnet 4.6 (copilot)']     # Model priority list
user-invocable: true                        # Show in dropdown
disable-model-invocation: false             # Allow as subagent
handoffs:                                   # Transition buttons
  - label: Next Step
    agent: OtherAgent
    prompt: Continue with the next phase.
    send: false                             # false = user reviews first
---

# Instructions in Markdown

Your agent's system prompt and guidelines go here.
Reference tools with #tool:toolName syntax.
```

---

## Summary

| Concept         | What it does                                                     |
|-----------------|------------------------------------------------------------------|
| Agent Sessions  | Unified dashboard for all running agents                         |
| Custom Agents   | `.agent.md` files with specialized roles and tool access         |
| Subagents       | Context-isolated workers for parallel subtasks                   |
| Handoffs        | Guided transitions between agents in a workflow                  |
| Skills          | Portable instruction folders loaded on-demand                    |
| MCP Servers     | External tool integrations (databases, APIs, services)           |
| AGENTS.md       | Cross-agent instruction file for consistent behavior             |

The multi-agent paradigm in VS Code transforms AI-assisted development from a single-assistant model into a coordinated team of specialized agents — each with the right tools, context, and model for the job.
