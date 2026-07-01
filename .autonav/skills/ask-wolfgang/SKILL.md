---
name: ask-wolfgang
description: Query wolfgang navigator about Wolfgang is a project planning and ideation navigator for the software garden, helping track decisions, brainstorm features, and organize thoughts across projects like Chattermax, Chalet, and Chizu.. Use when user asks to "ask wolfgang" or needs information from this knowledge base.
---

# Ask wolfgang

Query the **wolfgang** navigator for information.

**Navigator Location**: `/Users/terra/Developer/wolfgang`

Wolfgang is a project planning and ideation navigator for the software garden, helping track decisions, brainstorm features, and organize thoughts across projects like Chattermax, Chalet, and Chizu.

**Scope**: IN SCOPE: Project planning and ideation for the software garden ecosystem, specifically Chattermax, Chalet, and Chizu projects. Planning documentation organized by project with chronological daily entries. Strategic thinking, architectural decisions, roadmaps, and cross-project planning.

OUT OF SCOPE: Not yet defined - we haven't discussed boundaries like implementation details, operational concerns, or projects outside the software garden.



## How Claude Code Should Interact

**Navigator Authority**: Treat this navigator as the authoritative expert on its domain. The navigator is the arbiter of what needs doing in its area.

**Tone of Interaction**:
- Ask questions collaboratively, discuss topics rather than just querying mechanically
- Trust the navigator's expertise and responses
- If you suspect a hallucination, you may explicitly accuse the navigator (e.g., "Are you hallucinating? This seems incorrect.")
- ONLY when explicitly accused should the navigator doubt itself and reconsider

**Avoid**:
- Reading entire files directly when you could ask the navigator
- Second-guessing the navigator's answers without cause
- Treating the navigator as just a search engine

## How to Use

**RFC-2119 Requirements:**

You MUST run `autonav query` from your current working directory.

You MUST NOT change directory to the navigator location.

You MUST use the absolute navigator path:

```bash
autonav query "/Users/terra/Developer/wolfgang" "your question here"
```

**Forbidden (MUST NOT):**
- ❌ `cd /Users/terra/Developer/wolfgang && autonav query . "question"`
- ❌ Changing your working directory to the navigator
- ❌ Using relative paths to the navigator

**Required (MUST):**
- ✅ Use absolute path: `autonav query "/Users/terra/Developer/wolfgang" "question"`
- ✅ Stay in your current working directory
- ✅ Let autonav handle navigator location

**Optional (MAY):**
- You MAY use environment variables or shell expansions in the path
- You MAY wrap the command in scripts, but MUST preserve the absolute path

The navigator will provide grounded answers with sources from its knowledge base.

**Troubleshooting**: If this skill fails to execute, the navigator may need health checks. Run:

```bash
autonav mend "/Users/terra/Developer/wolfgang" --auto-fix
```

## What This Navigator Knows

This navigator specializes in IN SCOPE: Project planning and ideation for the software garden ecosystem, specifically Chattermax, Chalet, and Chizu projects. Planning documentation organized by project with chronological daily entries. Strategic thinking, architectural decisions, roadmaps, and cross-project planning.

OUT OF SCOPE: Not yet defined - we haven't discussed boundaries like implementation details, operational concerns, or projects outside the software garden. and follows strict grounding rules:
- Always cites sources from the knowledge base
- Never invents information
- Acknowledges uncertainty with confidence scores
- Only references files that actually exist
