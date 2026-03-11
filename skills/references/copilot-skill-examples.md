# Copilot Skill Examples

Two real-world examples showing the AI-driven and shell-based patterns.

## AI-Driven Example: readme skill

The readme skill generates README.md files for aux4 packages. It uses `aux4 ai agent ask` to spawn an inner LLM agent that reads the `.aux4` file, understands the package, and generates documentation.

### package/.aux4

```json
{
  "scope": "agent",
  "name": "copilot-skill-readme",
  "version": "0.1.0",
  "license": "Apache-2.0",
  "description": "Copilot skill for generating README files",
  "dependencies": [
    "aux4/copilot"
  ],
  "profiles": [
    {
      "name": "copilot:skills",
      "commands": [
        {
          "name": "readme",
          "execute": ["profile:copilot:skills:readme"],
          "help": {
            "text": "Generate README.md for aux4 packages"
          }
        }
      ]
    },
    {
      "name": "copilot:skills:readme",
      "commands": [
        {
          "name": "prompt",
          "execute": [
            "cat ${packageDir}/instructions/readme.md"
          ],
          "help": {
            "text": "Get the prompt/instructions for this skill"
          }
        },
        {
          "name": "generate",
          "execute": [
            "set:question=read the `${aux4}` file and output the content for the `${path}` file. Do not save the file, just output the markdown content. ${message}",
            "aux4 ai agent ask --instructions ${packageDir}/instructions/readme.md param(question) > ${path}"
          ],
          "help": {
            "text": "Generate README.md file",
            "variables": [
              {
                "name": "aux4",
                "text": "The path to the .aux4 file",
                "default": ".aux4"
              },
              {
                "name": "path",
                "text": "The output path for README.md",
                "default": "README.md"
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

### Key Patterns

- **`copilot:skills` profile** — registers the skill with name `readme` and descriptive help text
- **`copilot:skills:readme` profile** — contains `prompt` (mandatory) and `generate` (action)
- **`prompt` command** — uses `cat` to output the instruction file contents
- **`generate` command** — uses `set:` to build a question string, then calls `aux4 ai agent ask` with `--instructions` pointing to the skill's instruction file. Output is redirected to the target file path.
- **`param(question)`** — passes the question as a positional argument (only included if non-empty)
- **Variables** — `aux4` (path to .aux4 file), `path` (output path), `message` (extra instructions) all have defaults
- **No `aux4/ai-agent` in dependencies** — it's available transitively through the copilot ecosystem. The skill only declares `aux4/copilot`.
- **No `main` profile** — this is an addon package, not a standalone CLI

## Shell-Based Example: web skill

The web skill searches the web and fetches content from URLs. It uses `aux4/browser` (a headless browser package) directly from shell commands — no inner LLM agent needed.

### package/.aux4

```json
{
  "scope": "agent",
  "name": "copilot-skill-web",
  "version": "0.1.0",
  "license": "Apache-2.0",
  "description": "Copilot skill for searching the web and fetching content from URLs",
  "dependencies": [
    "aux4/copilot",
    "aux4/browser"
  ],
  "profiles": [
    {
      "name": "copilot:skills",
      "commands": [
        {
          "name": "web",
          "execute": ["profile:copilot:skills:web"],
          "help": {
            "text": "Search the web and fetch content from URLs using a headless browser"
          }
        }
      ]
    },
    {
      "name": "copilot:skills:web",
      "commands": [
        {
          "name": "prompt",
          "execute": [
            "cat ${packageDir}/instructions/web.md"
          ],
          "help": {
            "text": "Get the prompt/instructions for this skill"
          }
        },
        {
          "name": "search",
          "execute": [
            "aux4 browser start --persistent true >/dev/null 2>&1 & sleep 3; SESSION=$(aux4 browser open --url \"https://search.yahoo.com/search?p=$(echo '${query}' | sed 's/ /+/g')\" 2>/dev/null); aux4 browser content --session $SESSION --format markdown; aux4 browser close --session $SESSION >/dev/null 2>&1; aux4 browser stop >/dev/null 2>&1"
          ],
          "help": {
            "text": "Search the web for information on a topic",
            "variables": [
              {
                "name": "query",
                "text": "The search query",
                "arg": true
              }
            ]
          }
        },
        {
          "name": "fetch",
          "execute": [
            "aux4 browser start --persistent true >/dev/null 2>&1 & sleep 3; SESSION=$(aux4 browser open --url \"${url}\" 2>/dev/null); aux4 browser content --session $SESSION --format markdown; aux4 browser close --session $SESSION >/dev/null 2>&1; aux4 browser stop >/dev/null 2>&1"
          ],
          "help": {
            "text": "Fetch and extract content from a URL",
            "variables": [
              {
                "name": "url",
                "text": "The URL to fetch",
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

### Key Patterns

- **Dependencies** — `aux4/copilot` plus `aux4/browser` (the external tool). No `aux4/ai-agent` needed.
- **`prompt` command** — same pattern as AI-driven: `cat` the instruction file
- **`search` and `fetch` commands** — pure shell commands that start a headless browser, open a URL, extract content as markdown, then clean up. All output goes to stdout.
- **Raw output** — the copilot receives the raw markdown content from the browser and interprets/summarizes it for the user. The skill itself doesn't need an LLM.
- **`arg: true`** — `query` and `url` are positional arguments, not named flags
- **No `main` profile** — addon package pattern

### Comparing the Two Patterns

| Aspect | readme (AI-driven) | web (shell-based) |
|--------|--------------------|--------------------|
| Action executor | `aux4 ai agent ask` | Shell commands |
| Needs LLM in action? | Yes (inner agent) | No |
| Tool access | readFile, writeFile, etc. | None (shell only) |
| Output | Agent generates final result | Raw stdout for copilot to interpret |
| Cost per action | Higher (LLM tokens) | Lower (no LLM) |
| Dependencies | `aux4/copilot` | `aux4/copilot` + `aux4/browser` |
| Best for | File generation, code analysis | Wrapping CLI tools |
