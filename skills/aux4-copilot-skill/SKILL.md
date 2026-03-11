---
name: aux4-copilot-skill
description: Creates copilot skills — pluggable capabilities for the aux4 copilot with AI-driven or shell-based patterns.
user-invocable: true
disable-model-invocation: false
argument-hint: [skill-description]
---

# Create a Copilot Skill

Create a copilot skill package based on: $ARGUMENTS

## Overview

Copilot skills are pluggable capabilities that extend what the aux4 copilot can do. Each skill is a standalone aux4 package that registers itself under the `copilot:skills` profile. The copilot discovers installed skills automatically and matches user requests to skills based on their help text.

Skills live in `packages/ai/copilot-skill-<name>/` and follow a specific structure with a mandatory `prompt` command that returns the skill's instruction file.

## Two Skill Patterns

### AI-Driven Skills

Action commands call `aux4 ai agent ask` to spawn an inner LLM agent with access to tools (readFile, writeFile, editFile, listFiles, searchFiles, executeAux4, etc.). Use this pattern when the skill needs to read files, write files, make decisions, or interact with the user during execution.

**Examples:** readme, docs, test, planner

**Dependencies:** `aux4/copilot` (and optionally `aux4/ai-agent` if not already available transitively)

### Shell-Based Skills

Action commands run shell commands directly and return raw output to stdout. The copilot receives the output and interprets it for the user. This pattern is cheaper and more reliable for skills that wrap external tools — no inner LLM round-trips needed.

**Examples:** web (wraps `aux4/browser` to search and fetch URLs)

**Dependencies:** `aux4/copilot` plus the external tool package (e.g., `aux4/browser`). No `aux4/ai-agent` needed.

### Choosing a Pattern

| Question | AI-Driven | Shell-Based |
|----------|-----------|-------------|
| Needs to read/write files? | Yes | No |
| Needs to make decisions? | Yes | No |
| Wraps an external CLI tool? | No | Yes |
| Needs tool access (search, list, edit)? | Yes | No |
| Output needs LLM interpretation? | No (agent handles it) | Yes (copilot interprets) |

## Copilot Skill Package Structure

````
copilot-skill-<name>/
├── package/
│   ├── .aux4              # Profile registration + commands
│   ├── instructions/
│   │   └── <name>.md      # Instruction file(s)
│   ├── README.md
│   ├── LICENSE
│   ├── man/               # Man pages
│   └── test/              # Tests (.test.md)
├── .aux4                  # Build/dev commands (optional)
└── README.md
````

## .aux4 Configuration

### AI-Driven Template

```json
{
  "scope": "agent",
  "name": "copilot-skill-<name>",
  "version": "0.1.0",
  "license": "Apache-2.0",
  "description": "Copilot skill for <what it does>",
  "dependencies": [
    "aux4/copilot"
  ],
  "profiles": [
    {
      "name": "copilot:skills",
      "commands": [
        {
          "name": "<skill-name>",
          "execute": ["profile:copilot:skills:<skill-name>"],
          "help": {
            "text": "What this skill does — shown in copilot skills --help"
          }
        }
      ]
    },
    {
      "name": "copilot:skills:<skill-name>",
      "commands": [
        {
          "name": "prompt",
          "execute": [
            "cat ${packageDir}/instructions/<name>.md"
          ],
          "help": {
            "text": "Get the prompt/instructions for this skill"
          }
        },
        {
          "name": "<action>",
          "execute": [
            "aux4 ai agent ask --instructions ${packageDir}/instructions/<name>.md --question '<task description with ${variables}>'"
          ],
          "help": {
            "text": "Action description",
            "variables": [
              {
                "name": "aux4",
                "text": "The path to the .aux4 file",
                "default": ".aux4"
              },
              {
                "name": "message",
                "text": "Additional instructions",
                "default": ""
              }
            ]
          }
        }
      ]
    }
  ]
}
```

### Shell-Based Template

```json
{
  "scope": "agent",
  "name": "copilot-skill-<name>",
  "version": "0.1.0",
  "license": "Apache-2.0",
  "description": "Copilot skill for <what it does>",
  "dependencies": [
    "aux4/copilot",
    "aux4/<external-tool>"
  ],
  "profiles": [
    {
      "name": "copilot:skills",
      "commands": [
        {
          "name": "<skill-name>",
          "execute": ["profile:copilot:skills:<skill-name>"],
          "help": {
            "text": "What this skill does — shown in copilot skills --help"
          }
        }
      ]
    },
    {
      "name": "copilot:skills:<skill-name>",
      "commands": [
        {
          "name": "prompt",
          "execute": [
            "cat ${packageDir}/instructions/<name>.md"
          ],
          "help": {
            "text": "Get the prompt/instructions for this skill"
          }
        },
        {
          "name": "<action>",
          "execute": [
            "<shell commands that run the external tool and output results to stdout>"
          ],
          "help": {
            "text": "Action description",
            "variables": [
              {
                "name": "query",
                "text": "The input",
                "arg": true
              }
            ]
          }
        }
      ]
    }
  ]
}
```

### Key Configuration Rules

- **No `main` profile** — copilot skill packages are addons, not standalone CLIs. They only have `copilot:skills` and `copilot:skills:<name>` profiles.
- **`prompt` command is mandatory** — the copilot calls `aux4 copilot skills <name> prompt` to read the skill's instructions before deciding how to use it.
- **Help text is the discovery mechanism** — see "Making Skills Discoverable" below.
- **Dependencies** — always include `aux4/copilot`. Add `aux4/ai-agent` only if the skill needs it and it's not already available. Shell-based skills add their external tool package instead.

## Making Skills Discoverable

### How Discovery Works

At startup, the copilot runs `aux4 copilot skills --help` and injects the output into its system prompt under an `## Installed Skills` heading. The LLM then semantically matches the user's request against each skill's help text. That one-liner in the `copilot:skills` profile is the **only signal** the LLM has for choosing a skill — there is no scoring algorithm, no tags, no embeddings. The entire discoverability surface is that single `help.text` string.

```text
User request → LLM reads "Installed Skills" help texts → semantic match → runs prompt → executes skill
```

### Writing Effective Help Text

The `help.text` in your `copilot:skills` entry should:

1. **Start with an action verb** — tells the LLM what the skill *does*
2. **Include key nouns the user would say** — the LLM matches user intent to these words
3. **Cover synonyms and related terms** — if users might phrase the request differently, include alternative terms
4. **Be specific, not vague** — generic text competes with other skills and direct LLM action

**GOOD** — specific, action-oriented, includes key nouns:

```text
"Generate README.md for aux4 packages"
"Search the web and fetch content from URLs using a headless browser"
"Generate and fix tests for aux4 packages"
"Plan and execute multi-step tasks with structured task lists"
```

**BAD** — vague, missing key terms, too generic:

```text
"Handles documentation"           # what kind? generate? update? for what?
"Web utilities"                    # what does it do? search? scrape? browse?
"Testing helper"                   # write tests? run tests? fix tests?
"Helps with tasks"                 # every skill helps with tasks
```

### Tips

- Think about what the user would type: "generate a README", "search the web for X", "write tests for Y" — your help text should contain those same words
- If your skill does multiple things (e.g., write AND fix tests), mention both
- Include the domain or tool name if relevant (e.g., "using a headless browser", "for aux4 packages")
- Keep it to one sentence — the `--help` output is a flat list and long descriptions get truncated
- Test discoverability: run `aux4 copilot ask "your expected user request"` and verify the copilot selects your skill

## Writing Instruction Files

The instruction file is what the copilot reads (via the `prompt` command) to understand how to use the skill.

### Structure

````markdown
# Skill Name

Brief description of what this skill does.

## Commands

- `aux4 copilot skills <name> <action>` - What it does

Run `aux4 copilot skills <name> <action> --help` for parameter details.

## Task Description

Explain what the skill should accomplish step by step.

## CRITICAL RULES

**ALWAYS:**
- Do this specific thing
- Follow this format

**NEVER:**
- Do this wrong thing
- Make this common mistake

## Process

### Step 1: Gather Information
[What to read/check first]

### Step 2: Execute
[What to do]

### Step 3: Verify
[How to confirm success]

## Common Pitfalls

**WRONG** (will fail):
```text
some wrong example    # explanation
```

**CORRECT:**
```text
the right way         # explanation
```
````

### Best Practices

1. **Start with commands** — list available commands at the top so the copilot knows what tools it has
2. **Include explicit guardrails** — ALWAYS/NEVER rules prevent common mistakes
3. **Provide good/bad examples** — show what correct and incorrect output looks like
4. **Add token management guidance** — for skills that read many files, tell the agent to focus on essential files and not paste contents verbatim
5. **Add verification steps** — tell the agent how to confirm the output is correct
6. **Don't hardcode implementation details** — instruction files should describe *what* to do, not paste entire file contents

### For AI-Driven Skills

The instruction file teaches the inner LLM agent what to do. It should include:
- What files to read and in what order
- What to generate or modify
- Rules about output format
- How to use available tools (readFile, writeFile, editFile, searchFiles, etc.)

### For Shell-Based Skills

The instruction file teaches the copilot how to interpret the raw output from shell commands. It should include:
- What each command does and what its output looks like
- How to summarize/present the output to the user
- When to use each command

## Simple vs Multi-Step Skills

### Simple Skills

For tasks with a single action (generate a file, search something):

````
copilot-skill-<name>/
├── package/
│   ├── .aux4
│   └── instructions/
│       └── <name>.md           # Single instruction file
````

Commands: `prompt` + 1-2 action commands (e.g., `generate`, `write`, `fix`)

### Multi-Step Skills

For workflows requiring multiple stages:

````
copilot-skill-<name>/
├── package/
│   ├── .aux4
│   └── instructions/
│       ├── <name>.md           # Main workflow overview
│       ├── step1.md            # Step 1 instructions
│       ├── step2.md            # Step 2 instructions
│       └── step3.md            # Step 3 instructions
````

The main prompt file describes the overall workflow. Each step can have its own instruction file with detailed guidance. Multi-step skills may use sub-profiles with their own `prompt` commands.

### Context Passing Between Steps

- **File-based handoffs** — step 1 writes a file, step 2 reads it
- **Stdin piping** — use `--context true` to pipe output between commands
- **Config files** — use `--config` for structured data

## Complete Examples

Read `../references/copilot-skill-examples.md` for two complete real-world examples:

1. **AI-Driven: readme skill** — generates README.md files using `aux4 ai agent ask`
2. **Shell-Based: web skill** — searches and fetches web pages using `aux4/browser`

## Testing Your Skill

1. Verify the copilot discovers it:
   ```bash
   aux4 copilot skills --help
   ```

2. Test the prompt command:
   ```bash
   aux4 copilot skills <name> prompt
   ```

3. Test each action command:
   ```bash
   aux4 copilot skills <name> <action> --help
   ```

4. Test via the copilot:
   ```bash
   aux4 copilot ask "task that should trigger your skill"
   ```

5. Test that unrelated requests don't trigger the skill:
   ```bash
   aux4 copilot ask "unrelated question"
   ```

## Instructions

When creating a copilot skill:

1. Load `/aux4-package`, `/aux4-test`, and `/aux4-docs` skills first — copilot skills are aux4 packages and follow the same packaging conventions.
2. Ask the user which pattern to use: AI-driven (needs file access, decisions, user interaction) or shell-based (wraps an external tool).
3. Scaffold the package using `/aux4-package` conventions. The package lives in `packages/ai/copilot-skill-<name>/`.
4. Create the `package/.aux4` file with `copilot:skills` and `copilot:skills:<name>` profiles. Use the appropriate template above. Remember: no `main` profile.
5. The `prompt` command is mandatory — it must `cat` the instruction file.
6. Write the instruction file(s) in `package/instructions/`. Follow the structure and best practices above.
7. Make the help text in the `copilot:skills` profile descriptive — the copilot matches skills by help text.
8. Create man pages following `/aux4-docs` conventions. Man page filenames for copilot skill commands use `_` for `:` in profile names (e.g., profile `copilot:skills:web` command `search` → `copilot_skills_web__search.md`).
9. Create tests following `/aux4-test` conventions.
10. Create `package/README.md` following `/aux4-docs` conventions.
11. Build and install locally with `cd package && aux4 aux4 releaser install` to test.
12. Verify the skill appears in `aux4 copilot skills --help` and works end-to-end.
13. When writing markdown files (instructions, man pages, `.test.md`), use 4 backticks (````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
