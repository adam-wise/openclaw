# Safe Self-Teaching Agent Design

## How Agents "Learn" - Risk Levels

| Mechanism | What It Does | Risk |
|-----------|--------------|------|
| Memory files | Writes facts/preferences to markdown | ✅ Safe |
| Prompt libraries | Saves prompts that worked | ✅ Safe |
| Parameterized automations | Calls pre-defined workflows with params | ✅ Safe |
| Sandboxed code execution | Runs code in isolated container | ⚠️ Medium |
| Capability requests | Proposes new tools, human approves | ✅ Safe |
| Shell execution / file creation | Installs packages, creates scripts | 🔴 Dangerous |


## Layer 1: Knowledge Accumulation (Safe)

Agent learns *facts*, not *capabilities*:

```typescript
tool("learn_fact", { fact: string }, async ({ fact }) => {
  const today = `memory/${new Date().toISOString().slice(0, 10)}.md`;
  await fs.appendFile(today, `\n- ${fact}`);
  return { ok: true };
});

tool("learn_preference", { key: string, value: string }, async ({ key, value }) => {
  // Store in structured section of MEMORY.md
  await appendToSection("MEMORY.md", "Preferences", `- ${key}: ${value}`);
  return { ok: true };
});
```

Example MEMORY.md the agent builds:
```markdown
## Learned Patterns
- User prefers bullet points over paragraphs
- For daily summaries: check calendar first, then email
- The project uses pnpm, not npm

## Preferences
- timezone: America/Los_Angeles
- response_style: concise
```


## Layer 2: Prompt/Workflow Libraries (Safe)

Agent saves *prompts that work*, not *code that runs*:

```typescript
tool("prompt_save", { name: string, prompt: string }, async ({ name, prompt }) => {
  await db.run("INSERT INTO saved_prompts VALUES (?, ?)", [name, prompt]);
  return { ok: true };
});

tool("prompt_get", { name: string }, async ({ name }) => {
  return db.get("SELECT prompt FROM saved_prompts WHERE name = ?", [name]);
});

tool("workflow_save", { name: string, steps: string[] }, async ({ name, steps }) => {
  await db.run("INSERT INTO workflows VALUES (?, ?)", [name, JSON.stringify(steps)]);
  return { ok: true };
});
```

Agent learns: "When user asks for X, use this prompt/workflow."


## Layer 3: Parameterized Automations (Safe)

Pre-define automation templates, agent chooses which + params:

```typescript
const AUTOMATIONS = {
  "daily_summary": async ({ date, include_calendar, include_email }) => { /* fixed code */ },
  "weekly_report": async ({ week_of, format }) => { /* fixed code */ },
  "inbox_triage": async ({ limit, priority_only }) => { /* fixed code */ },
};

tool("automation_run", { name: string, params: object }, async ({ name, params }) => {
  if (!AUTOMATIONS[name]) throw new Error("Unknown automation");
  return AUTOMATIONS[name](params);  // Can't create new ones, only call existing
});
```


## Layer 4: Sandboxed Code Execution (Medium Risk)

If agent needs to run dynamic code, isolate it completely:

```typescript
tool("compute", { code: string, language: "python" | "javascript" }, async ({ code, language }) => {
  const result = await runInSandbox({
    code,
    language,
    constraints: {
      network: false,        // No network access
      filesystem: "none",    // No file access
      memory: "128mb",       // Memory limit
      timeout: 10000,        // 10 second max
      allowedModules: []     // No imports
    }
  });
  return result.stdout;
});
```

**Can do**: calculations, data transformation, text processing
**Cannot do**: install packages, access files, make requests, persist anything


## Layer 5: Capability Requests (Human-in-Loop)

Agent *proposes*, you *approve and implement*:

```typescript
tool("request_capability", {
  name: string,
  description: string,
  example_use: string
}, async (request) => {
  await db.run(
    "INSERT INTO capability_requests VALUES (?, ?, ?, 'pending')",
    [request.name, request.description, request.example_use]
  );
  return "Request logged. Human will review and implement if approved.";
});
```

You periodically review requests, manually add tools that make sense.


## What NOT To Build

```typescript
// ❌ NEVER expose these
tool("shell_exec", { command: string }) → // NO
tool("file_write", { path: any, content }) → // NO (except specific memory paths)
tool("install_package", { package: string }) → // NO
tool("create_tool", { name, code }) → // NO
tool("fetch_url", { url: any }) → // NO (or heavily restrict)
```


## Summary

| Learning Type | Implementation | Risk |
|---------------|----------------|------|
| Remember facts | Append to memory/*.md | ✅ Safe |
| Remember preferences | Structured MEMORY.md sections | ✅ Safe |
| Learn what works | Save prompts/workflows to DB | ✅ Safe |
| Choose automations | Call pre-defined functions with params | ✅ Safe |
| Process data dynamically | Sandboxed code (no network/fs) | ⚠️ Medium |
| Request new capabilities | Log request, human implements | ✅ Safe |
| Self-modify / install | DON'T DO THIS | 🔴 Dangerous |

**90% of useful "learning" is Layers 1-3.** The dangerous stuff is rarely needed.
