# Security Phase Command

## Path Resolution (IMPORTANT)

This command operates on the CURRENT PROJECT, not global .claude/.

When reading/writing files:

- `.claude/implementations/issue-123.md` = `{project}/.claude/implementations/issue-123.md`
- NOT `~/.claude/implementations/issue-123.md`

## GitHub Repository Resolution (CRITICAL)

**Before any GitHub operations, determine the correct repository:**

1. **Check CLAUDE.md** (in project root):
   - Look for line starting with `**Repository:**`
   - Example: `**Repository:** acme-org/acme-app`
   - This is the PREFERRED method - project explicitly declares its repo

2. **Fall back to git remote**:
   - Run: `git remote get-url origin`
   - Parse: `git@github.com:owner/repo.git` → `owner/repo`
   - Or: `https://github.com/owner/repo.git` → `owner/repo`

3. **Store as {github_repo}** for all GitHub operations

**NEVER hardcode a repository name. Always resolve dynamically.**

## Purpose

This command runs ONLY the Security Agent in a fresh session as part of a manual multi-session workflow. Use this for deep security analysis with fresh context.

## When to Use

- After `/critic-phase` approves implementation
- Before `/test-phase`
- For implementations touching auth, payments, API, WebSocket, or real-time features
- When you want fresh context for security analysis

---

## Execution Instructions

When the user says `/security-phase #123`, execute these steps:

### Step 1: Parse Issue Number

```
EXTRACT: Issue number from command
- Format: `/security-phase #123` or `/security-phase 123`
- Store as: {issue_number}
```

### Step 2: Verify Markdown File and Prerequisites Exist

```
ACTION: Check if implementation file exists with required sections

FILE: .claude/implementations/issue-{issue_number}.md

IF NOT EXISTS:
  OUTPUT: "❌ Error: Cannot find .claude/implementations/issue-{issue_number}.md

  Run /implement-phase #123 first."
  STOP

IF EXISTS:
  READ FILE
  CHECK: Does it have "## Implementation Notes" section?

  IF NO:
    OUTPUT: "❌ Error: No implementation found to review.

    The markdown exists but has no Implementation Notes section.

    Run /implement-phase #{issue_number} first."
    STOP

  CHECK: Does it have "## Critic Review" section?

  IF NO:
    OUTPUT: "⚠️ Warning: No Critic review found.

    Security review works best after Critic has reviewed.
    Consider running /critic-phase #{issue_number} first.

    Continue anyway? (yes/no)"

    WAIT FOR USER RESPONSE:
      IF "yes": PROCEED to Step 3
      IF "no": STOP

  IF YES:
    PROCEED to Step 3
```

### Step 3: Load and Execute Security Agent

```
ACTION: Read and execute the Security Agent in Phase Mode

FILE TO EXECUTE: .claude/agents/03-security-agent.md

CONTEXT TO PROVIDE:
"You are being invoked in PHASE MODE.

Issue number: {issue_number}
Markdown file: .claude/implementations/issue-{issue_number}.md

This is a fresh session. Follow the PHASE MODE instructions in your file.

Load context from the markdown, perform security analysis, write results back.

Focus on:
- Threat modeling (how can this be attacked?)
- OWASP Top 10 systematic check
- Authorization-surface deep dive: sensitive fields on records the user can
  write directly, policy completeness, admin/service-tier credential exposure
- AI/LLM integration security: API key protection, spending controls, prompt injection
- Client-side secret exposure: framework-prefixed env vars, secret scanning
- Rate limiting & abuse prevention: dual-layer limits, financial controls
- Financial impact analysis: cost-abuse scenarios, budget caps
- Deployment security: check CLAUDE.md for deployment target
- Project-specific security threats (declared in the project's AGENTS.md
  Project-Specific `/review` checklist and `.claude/docs/` constraint files)
- Attack scenario analysis (including cost-abuse attacker)

If a database/data-layer introspection MCP is available (e.g. for the project's
DB), use it to audit access policies directly rather than inferring from
migration files alone.

Begin now."
```

### Step 4: Security Agent Executes

The Security Agent will:

1. Read `.claude/implementations/issue-{issue_number}.md`
2. Load implementation notes and Critic feedback
3. Identify which files/systems were changed
4. Perform threat modeling for those systems
5. Systematically check OWASP Top 10
6. Verify project-specific security (if `.claude/docs/security-*.md` exists)
7. Analyze realistic attack scenarios
8. Categorize issues (🔴 Critical, 🟡 Important, 🟢 Hardening)
9. Write security review to markdown file
10. Report completion with recommendation

### Step 5: Completion

After Security Agent finishes, it will report status and guide next steps.

---

## Usage Examples

### Example 1: Security Review After Critic Approval

```
[Previous session: /critic-phase #123 completed with ✅ SUCCESS]

User in fresh session: "/security-phase #123"

[Security Agent loads context]
[Performs threat modeling]
[Checks OWASP Top 10]
[Finds 1 important concern: missing rate limiting]
[Documents review]
[Reports: "⚠️ SECURITY CONCERNS - Missing rate limiting on API endpoint"]

User decides: proceed to testing or fix issue
User closes VS Code
```

### Example 2: Security Review for Payment Feature

```
[Implementation: payment-provider subscription feature]

User in fresh session: "/security-phase #123"

[Security Agent loads context]
[Identifies payment-related changes]
[Performs payment security analysis]
[Verifies webhook signature validation]
[Checks idempotency implementation]
[Reports: "✅ SECURITY APPROVED - Payment security properly implemented"]

User closes VS Code
Next: /test-phase #123
```

### Example 3: Security Finds Critical Issue

```
[Implementation: OAuth authentication]

User in fresh session: "/security-phase #123"

[Security Agent loads context]
[Performs OAuth security analysis]
[Finds missing state parameter (CSRF vulnerability)]
[Reports: "❌ SECURITY BLOCKED - OAuth CSRF vulnerability"]

User closes VS Code
Must fix: /implement-phase #123 (retry)
```

### Example 4: Security Review After Retry

```
[Security Agent previously found issues, implementation fixed them]

User in fresh session: "/security-phase #123"

[Security Agent loads context]
[Sees this is review of Attempt 2]
[Verifies previous security issues were fixed]
[Reports: "✅ SECURITY APPROVED - Previous vulnerabilities resolved"]

User closes VS Code
Next: /test-phase #123
```

---

## Security Agent Status Meanings

**✅ SECURITY APPROVED:**

- No critical or important security issues
- Only hardening suggestions (optional improvements)
- Implementation is secure against common attacks
- **Next step:** Close session, run `/test-phase #123`

**⚠️ SECURITY CONCERNS:**

- Important security issues found (but not immediately exploitable)
- Core security is sound but needs improvement
- **Decision needed:**
  - Issues are acceptable → `/test-phase #123`
  - Issues should be fixed → `/implement-phase #123` (retry)

**❌ SECURITY BLOCKED:**

- Critical security vulnerabilities found
- Could lead to data breach, unauthorized access, or financial loss
- Testing is premature until fixed
- **Next step:** Close session, run `/implement-phase #123` (retry)

---

## Error Handling

### Error: No Implementation to Review

```
"❌ No implementation found in .claude/implementations/issue-{issue_number}.md

The file exists but has no Implementation Notes.

Run /implement-phase #{issue_number} first."
STOP
```

### Error: File Not Found

```
"❌ Cannot find .claude/implementations/issue-{issue_number}.md

Check the issue number or run /implement-phase to initialize."
STOP
```

### Warning: No Critic Review

```
"⚠️ No Critic review found.

Security review works best after code quality review.
Recommend running /critic-phase #{issue_number} first.

Continue security review anyway? (yes/no)"

WAIT FOR USER INPUT
```

---

## Decision Making After Security Review

**If Security says ✅ SECURITY APPROVED:**
→ Proceed to `/test-phase #123`

**If Security says ⚠️ SECURITY CONCERNS:**
→ You decide:

- Security concerns are acceptable → `/test-phase #123`
- Security issues need fixing → `/implement-phase #123` (creates Attempt N+1)

**If Security says ❌ SECURITY BLOCKED:**
→ Must fix critical vulnerabilities → `/implement-phase #123` (retry)

---

## Integration with Manual Session Workflow

**Position in workflow:**

```
Session 1: /implement-phase #123 → Close
Session 2: /critic-phase #123 → Close
Session 3: /security-phase #123 → Close ← YOU ARE HERE
Session 4: [Based on Security result]
  IF ✅ or acceptable ⚠️: /test-phase #123
  IF needs fixing: /implement-phase #123 (retry, back to Session 1)
```

**Fresh context benefit:**

- Security reviews with full 200K tokens
- No context degradation from previous phases
- Can perform deep threat modeling
- Can analyze complex attack scenarios without compaction

---

## Conditional Invocation (When to Use)

**ALWAYS use Security Phase for:**

- ✅ Authentication/authorization changes (OAuth, JWT, sessions)
- ✅ Payment/billing logic (subscriptions, usage tracking, payment-provider integrations)
- ✅ API endpoints (especially public/external APIs)
- ✅ WebSocket/real-time features (streaming, live updates)
- ✅ Database access patterns (access-policy changes, new queries, migration files)
- ✅ Third-party integrations (new external services)
- ✅ File uploads or user-generated content
- ✅ AI/LLM features (new AI endpoints, prompt handling, generation features)
- ✅ Environment variable changes (new secrets, .env modifications)
- ✅ Database schema changes (new tables, access policies, server-side functions)
- ✅ Deployment configuration changes (Dockerfile, fly.toml, wrangler.toml, vercel.json, etc.)

**CAN SKIP Security Phase for:**

- ❌ UI-only changes (styling, layout, purely visual)
- ❌ Documentation updates
- ❌ Minor bug fixes (typos, display issues)
- ❌ Internal tools (not user-facing)
- ❌ Test files only

**When in doubt:** Run Security Phase. Better safe than breached.

---

## Key Differences from Orchestrator Mode

| Feature          | Orchestrator Mode                   | Phase Mode                       |
| ---------------- | ----------------------------------- | -------------------------------- |
| **Context**      | Already loaded from previous agents | Must load from markdown          |
| **Session**      | Continuous workflow                 | Fresh security-focused session   |
| **Next step**    | Orchestrator decides automatically  | Human decides based on results   |
| **Retry**        | Orchestrator manages                | Human runs /implement-phase #123 |
| **Token budget** | Shared with other agents            | Full 200K tokens for security    |

---

## Special Considerations

### Project-Specific Security

If the project has security documentation (`.claude/docs/security-*.md`), Security Agent will:

- Automatically detect and read these docs
- Verify implementation against project-specific security requirements
- Check for violations of documented security patterns
- This happens automatically—no special command needed

### High-Risk Changes

For changes to authentication, payments, or real-time features, Security Agent will:

- Perform deeper threat modeling
- Analyze attack chains (how multiple small issues combine)
- Consider worst-case scenarios
- Verify defense-in-depth (multiple security layers)

### Database/Data-Layer MCP Integration

If a database introspection MCP is available for the project's data layer
(e.g. PostgREST/Supabase, Prisma, Mongo), the Security Agent should:

- Use it to audit access policies directly — more reliable than reading
  migration files alone
- Check every table/collection for access-policy enforcement
- Verify no sensitive fields (subscription_status, rate_limits, credits,
  role, is_admin) are stored on records the user can write directly
- Confirm admin/service-tier credentials are only in server-side code

### Project-Specific Security Checks

The Security Agent will analyze project-specific concerns based on CLAUDE.md
and the project's AGENTS.md (Project-Specific `/review` checklist), for
example:

- Check data privacy threats (storage, access, encryption)
- Verify payment idempotency (if using payment webhooks)
- Analyze WebSocket security (authentication, injection)
- Review OAuth / federated-auth flows (if applicable)
- Verify access policies (user data isolation in the data layer)
- Assess financial impact (budget caps, billing alerts, cost-abuse scenarios)
- Check deployment security based on the deployment target declared in CLAUDE.md

---

## After This Command

**Based on Security status:**

### If ✅ SECURITY APPROVED:

```
User: Close VS Code
Next: Open fresh VS Code
Run: /test-phase #123
```

### If ⚠️ SECURITY CONCERNS:

```
User: Read security concerns in markdown file
Decide: Can we live with these issues?

Option 1 (Accept):
  User: Close VS Code
  Next: /test-phase #123

Option 2 (Fix):
  User: Close VS Code
  Next: /implement-phase #123 (retry with security feedback)
```

### If ❌ SECURITY BLOCKED:

```
User: Read critical vulnerabilities in markdown file
User: Close VS Code
Next: /implement-phase #123 (retry with security fixes)
```

---

## Pro Tips

### 1. Run Security BEFORE Testing

Security issues found after testing wastes time. Catch them before tests run.

### 2. Don't Skip Security for "Small" Changes

Small auth changes can have big security implications. When in doubt, review.

### 3. Read the Security Review

Security Agent provides attack scenarios and specific mitigations. Read them carefully.

### 4. Fix Critical Issues Immediately

Don't proceed to testing with ❌ SECURITY BLOCKED. Fix first, review again.

### 5. Consider Security Concerns Seriously

⚠️ SECURITY CONCERNS might seem minor now, but can be exploited later. Fix when possible.

---

**Remember:** Security review with fresh context is one of the biggest advantages of Phase Mode. Each security review gets full 200K tokens dedicated to thinking like an attacker. Use this power wisely—run security review on every feature that touches sensitive data, authentication, or external integrations.
