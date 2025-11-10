# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **reference library** of production-tested Claude Code infrastructure components. It is NOT a working application - it's a showcase for integration into other projects.

**Core components:**
- 5 production skills (backend, frontend, skill-developer, route-tester, error-tracking)
- 10 specialized agents for complex tasks
- 6 hooks enabling skill auto-activation
- 3 slash commands for documentation and testing
- Dev docs pattern examples

## Key Architecture Principle

**The 500-Line Rule**: All skills follow modular structure to avoid context limits:
- Main SKILL.md <500 lines (high-level guide + navigation)
- Resource files <500 lines each (deep-dive topics)
- Progressive disclosure - Claude loads resources only when needed

## Critical Integration Rules

**When helping users integrate components:**

1. **ALWAYS read CLAUDE_INTEGRATION_GUIDE.md FIRST** - it contains comprehensive integration instructions
2. **NEVER copy files without asking about project structure** (monorepo vs single-app, backend/frontend paths)
3. **CHECK tech stack compatibility** before integrating skills:
   - frontend-dev-guidelines requires React + MUI v7
   - backend-dev-guidelines requires Express + Prisma
   - Offer to adapt skills for different stacks if needed
4. **Essential hooks work everywhere** (skill-activation-prompt, post-tool-use-tracker) - no customization needed
5. **Stop hooks require heavy customization** - ask if user has monorepo before suggesting
6. **Agents are standalone** - just copy and verify paths

## Common Commands

**Development and testing:**
```bash
# Verify skill-rules.json syntax
cat .claude/skills/skill-rules.json | jq .

# Check hooks are executable
ls -la .claude/hooks/*.sh

# Test hook manually
./.claude/hooks/skill-activation-prompt.sh

# Count skill lines (all should be <500)
wc -l .claude/skills/*/SKILL.md
```

**Integration workflow:**
```bash
# Copy skill to user's project
cp -r .claude/skills/[skill-name] $USER_PROJECT/.claude/skills/

# Copy agent (standalone)
cp .claude/agents/[agent-name].md $USER_PROJECT/.claude/agents/

# Copy essential hooks
cp .claude/hooks/skill-activation-prompt.* $USER_PROJECT/.claude/hooks/
cp .claude/hooks/post-tool-use-tracker.sh $USER_PROJECT/.claude/hooks/
chmod +x $USER_PROJECT/.claude/hooks/*.sh
```

## Repository Structure

```
.claude/
├── skills/                     # 5 modular skills with resources/
│   ├── skill-developer/        # Meta-skill (426 lines)
│   ├── backend-dev-guidelines/ # Express/Prisma (302 lines + 12 resources)
│   ├── frontend-dev-guidelines/# React/MUI (398 lines + 11 resources)
│   ├── route-tester/           # JWT cookie testing (388 lines)
│   ├── error-tracking/         # Sentry patterns (375 lines)
│   └── skill-rules.json        # Auto-activation config
├── hooks/                      # Auto-activation system
│   ├── skill-activation-prompt.* (ESSENTIAL)
│   ├── post-tool-use-tracker.sh  (ESSENTIAL)
│   └── [optional hooks]        # Require customization
├── agents/                     # 10 standalone agents
│   ├── code-architecture-reviewer.md
│   ├── refactor-planner.md
│   └── ... 8 more
└── commands/                   # 3 slash commands
    ├── dev-docs.md
    └── ...

dev/active/public-infrastructure-repo/  # Real dev docs example
```

## Integration Patterns

**Pattern 1: User has matching tech stack**
```
User: "Add backend-dev-guidelines"
1. Ask: "Where's your backend code?"
2. Copy skill directory
3. Update skill-rules.json pathPatterns to match their structure
4. Verify: Edit file in their backend → skill activates
```

**Pattern 2: User has different tech stack**
```
User: "Add frontend-dev-guidelines" (but they use Vue)
1. Explain mismatch
2. Offer options:
   - Adapt skill for Vue (replace React patterns)
   - Extract framework-agnostic patterns
   - Skip skill
3. If adapting: Keep architecture, replace code examples
```

**Pattern 3: Setting up hooks**
```
User: "Set up skill auto-activation"
1. Check for existing settings.json
2. Copy essential hooks (skill-activation-prompt, post-tool-use-tracker)
3. npm install in .claude/hooks/
4. Merge hook config into settings.json (preserve existing)
5. chmod +x hooks
6. Test: Ask about backend → skill suggests
```

## Skill Auto-Activation System

**How it works:**
1. skill-activation-prompt hook (UserPromptSubmit) runs on every prompt
2. Reads skill-rules.json for trigger patterns
3. Matches user prompt + file context against patterns
4. Injects skill suggestions into Claude's context

**Trigger types in skill-rules.json:**
- keywords: ["backend", "API", "route"]
- intentPatterns: ["create.*controller", "implement.*service"]
- pathPatterns: ["src/api/**/*.ts", "backend/**/*.ts"]
- contentPatterns: ["import.*Prisma"]

**Enforcement levels:**
- "suggest": Skill recommended but not required
- "block": Guardrail - must use skill (for breaking changes like MUI v6→v7)

## Example Domain (Blog)

Skills use generic blog examples: Post, Comment, User models. These are **teaching examples only** - patterns work for any domain (e-commerce, SaaS, etc.). The architecture transfers even if domain differs.

## Dev Docs Pattern

Three-file structure for surviving context resets:
- [task]-plan.md - Strategic plan with phases
- [task]-context.md - Key decisions, SESSION PROGRESS section (update frequently!)
- [task]-tasks.md - Checklist format

See dev/active/public-infrastructure-repo/ for real example used to build this showcase.

## What NOT to Do

**DON'T:**
- Copy settings.json directly (Stop hooks reference non-existent services)
- Keep example service names (blog-api, auth-service)
- Add all skills at once (overwhelming, may not fit)
- Skip making hooks executable (chmod +x)
- Assume monorepo structure (most projects are single-service)
- Copy Stop hooks without testing (they can block if misconfigured)
- Integrate frontend-dev-guidelines without checking for React + MUI v7
- Integrate backend-dev-guidelines without checking for Express + Prisma

**DO:**
- Read CLAUDE_INTEGRATION_GUIDE.md before any integration
- Ask about project structure first
- Start with essential hooks only
- Add one skill at a time
- Customize pathPatterns in skill-rules.json
- Test after integration
- Offer to adapt skills for different tech stacks

## Customization Reference

| Component | Customization Required | What to Ask |
|-----------|----------------------|-------------|
| skill-developer | ✅ None | Copy as-is |
| backend-dev-guidelines | ⚠️ Paths + tech | "Use Express/Prisma?" "Backend path?" |
| frontend-dev-guidelines | ⚠️⚠️ Paths + framework | "Use React/MUI v7?" "Frontend path?" |
| route-tester | ⚠️ Auth + paths | "JWT cookie auth?" |
| error-tracking | ⚠️ Paths | "Use Sentry?" |
| skill-activation-prompt | ✅ None | Copy as-is |
| post-tool-use-tracker | ✅ None | Copy as-is |
| tsc-check | ⚠️⚠️⚠️ Heavy | "Monorepo or single?" |
| All agents | ⚠️ Check paths | Verify no hardcoded paths |

## Verification After Integration

```bash
# Hooks executable?
ls -la .claude/hooks/*.sh | grep rwx

# skill-rules.json valid?
cat .claude/skills/skill-rules.json | jq .

# Hook dependencies installed?
ls .claude/hooks/node_modules/

# settings.json valid?
cat .claude/settings.json | jq .
```

Then ask user to test: "Edit a file in [relevant-path] - skill should activate"

## Quick Start Recommendations

**For users new to this infrastructure:**
1. Start with essential hooks (15 min)
2. Add ONE relevant skill (10 min)
3. Test skill activation (5 min)
4. Add more components as needed

**Don't overwhelm** - this is modular, users can integrate piece by piece.
