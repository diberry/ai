# Agent Component Audit Prompts

Use these prompts to validate whether files in your `.github/copilot/` directories are correctly placed based on their content and intended usage.

---

## Single File Audit Prompt

Use this prompt to audit one file at a time:

```
Please audit the following file and determine if it's in the correct directory based on its content and intended usage:

File: [FILE_PATH]

Content:
[PASTE FILE CONTENT HERE]

Based on these criteria, determine if this file is correctly placed:

📋 INSTRUCTIONS (.github/copilot/instructions/)
- Should be included with EVERY prompt automatically
- Contains: Project architecture, coding standards, domain knowledge, persistent context
- Always loaded regardless of user action
- Examples: project-context.md, coding-standards.md, architecture.md

💬 PROMPTS (.github/copilot/prompts/)
- User triggers manually when needed
- Contains: Reusable templates for repetitive tasks
- On-demand, user-initiated
- Examples: write-tests.md, generate-docs.md, refactor-code.md

🤖 CUSTOM AGENTS (.github/copilot/agents/)
- Defines specific multi-step workflows
- Contains: Specialized behavior patterns, domain-specific processes
- Task-specific automation
- Examples: code-reviewer.md, migration-assistant.md, security-auditor.md

⚡ SKILLS (.github/copilot/skills/)
- Provides discrete technical capabilities
- Contains: API integrations, external tool connections, code analyzers
- Capability that agents or prompts can invoke
- Examples: api-integration.ts, database-query.ts, deployment-helper.ts

Decision Tree:
1. Does this need to be loaded with EVERY prompt? → Instructions
2. Is this a reusable template users manually trigger? → Prompts
3. Does this define a specialized multi-step workflow? → Custom Agents
4. Does this provide a discrete technical capability? → Skills

Please provide:
1. ✅ CORRECT LOCATION or ❌ INCORRECT LOCATION
2. Current directory vs. Recommended directory
3. Reasoning based on content analysis
4. Key indicators from the file that led to this determination
```

---

## Batch Audit Prompt

Use this prompt to audit multiple files at once:

```
Please audit all files in the following .github/copilot/ directories and identify any that are misplaced:

[LIST FILES WITH THEIR CURRENT LOCATIONS]

For each file, determine if it should be in:
- instructions/ (always loaded, foundation knowledge)
- prompts/ (user-triggered templates)
- agents/ (specialized workflows)
- skills/ (technical capabilities)

Provide a migration plan table:

| Current Location | File | Recommended Location | Reason | Priority |
|-----------------|------|---------------------|---------|----------|
| prompts/ | example.md | instructions/ | Contains project-wide context | High |

Use this decision logic:
1. Always loaded for every prompt → instructions/
2. User manually triggers for specific tasks → prompts/
3. Defines multi-step workflow/behavior → agents/
4. Provides technical capability/integration → skills/
```

---

## Quick Reference: Component Characteristics

| Component | Auto-Loaded? | User Action Required? | Primary Purpose |
|-----------|--------------|----------------------|-----------------|
| **Instructions** | ✅ Yes (always) | ❌ No | Foundation & permanent context |
| **Prompts** | ❌ No | ✅ Yes (manual trigger) | Reusable task templates |
| **Custom Agents** | ❌ No | ✅ Yes (when workflow needed) | Specialized multi-step workflows |
| **Skills** | ❌ No | ✅ Yes (when capability needed) | Technical tools & integrations |

---

## Common Misplacement Patterns

### ❌ Wrong: Prompt file in instructions/
**Problem:** File contains a specific task template but is in instructions/  
**Fix:** Move to prompts/ - it should be user-triggered, not always loaded

### ❌ Wrong: Project standards in prompts/
**Problem:** Coding standards file is in prompts/ but should always apply  
**Fix:** Move to instructions/ - this is foundation knowledge needed everywhere

### ❌ Wrong: API integration in agents/
**Problem:** File provides a technical capability but is in agents/  
**Fix:** Move to skills/ - this is a discrete capability, not a workflow

### ❌ Wrong: Multi-step workflow in skills/
**Problem:** File orchestrates multiple steps but is in skills/  
**Fix:** Move to agents/ - this defines a specialized workflow pattern
