---
name: update-wolfgang
description: Update wolfgang navigator's documentation and knowledge base. Use when reporting implementation progress, documenting issues, or updating knowledge about Wolfgang is a project planning and ideation navigator for the software garden, helping track decisions, brainstorm features, and organize thoughts across projects like Chattermax, Chalet, and Chizu..
---

# Update wolfgang

Update the **wolfgang** navigator's documentation and knowledge base.

**Navigator Location**: `/Users/terra/Developer/wolfgang`

Wolfgang is a project planning and ideation navigator for the software garden, helping track decisions, brainstorm features, and organize thoughts across projects like Chattermax, Chalet, and Chizu.

**Scope**: IN SCOPE: Project planning and ideation for the software garden ecosystem, specifically Chattermax, Chalet, and Chizu projects. Planning documentation organized by project with chronological daily entries. Strategic thinking, architectural decisions, roadmaps, and cross-project planning.

OUT OF SCOPE: Not yet defined - we haven't discussed boundaries like implementation details, operational concerns, or projects outside the software garden.



## When to Use

Use this skill to:
- Report implementation progress or issues
- Update documentation after making changes
- Add new knowledge or learnings
- Document troubleshooting steps
- Create status reports or logs

## How to Use

**RFC-2119 Requirements:**

You MUST run `autonav update` from your current working directory.

You MUST NOT change directory to the navigator location.

You MUST use the absolute navigator path:

```bash
autonav update "/Users/terra/Developer/wolfgang" "your update message"
```

**Forbidden (MUST NOT):**
- ❌ `cd /Users/terra/Developer/wolfgang && autonav update . "message"`
- ❌ Changing your working directory to the navigator
- ❌ Using relative paths to the navigator

**Required (MUST):**
- ✅ Use absolute path: `autonav update "/Users/terra/Developer/wolfgang" "message"`
- ✅ Stay in your current working directory
- ✅ Let autonav handle navigator location

**Optional (MAY):**
- You MAY use environment variables or shell expansions in the path
- You MAY wrap the command in scripts, but MUST preserve the absolute path

**Troubleshooting**: If this skill fails to execute, the navigator may need health checks. Run:

```bash
autonav mend "/Users/terra/Developer/wolfgang" --auto-fix
```

**Example updates:**

Report progress:
```bash
autonav update "/Users/terra/Developer/wolfgang" "Implemented user authentication. Added OAuth2 flow with Google. See src/auth/ for details."
```

Document an issue:
```bash
autonav update "/Users/terra/Developer/wolfgang" "Database connection timeout issue: Increased pool size to 20 and added retry logic. Fixed in commit abc123."
```

Add learnings:
```bash
autonav update "/Users/terra/Developer/wolfgang" "Found that Lambda cold starts were causing 502s. Solution: provisioned concurrency of 5 instances. Response times now <100ms."
```
