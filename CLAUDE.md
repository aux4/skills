# aux4 - CLI Generator

aux4 is a CLI generator that creates high-level scripts using JSON-based `.aux4` configuration files.

## Workspace Structure

```
aux4/                          # Monorepo root
├── aux4/                      # Core CLI (Go)
├── pkger/                     # Package manager (Go)
├── packages/                  # aux4 packages
│   ├── config/                # config.yaml utility (Go)
│   ├── test/                  # .test.md test framework (Go)
│   ├── package-releaser/      # Build/publish/release tool
│   ├── encrypt/               # Encryption utility (Go)
│   ├── validator/             # Input validation (JS)
│   ├── db/                    # Database core (Go)
│   ├── db-mysql/              # MySQL adapter (JS)
│   ├── db-mssql/              # MSSQL adapter (JS)
│   ├── adapter/               # External adapters (JS)
│   ├── template/              # Template engine
│   ├── ai/                    # AI packages
│   │   ├── ai-agent/          # Core AI agent framework (JS/LangChain)
│   │   ├── copilot/           # AI copilot with pluggable skills
│   │   └── agent/             # Example agents (grammar, blog-writer)
│   └── ...                    # Other packages
├── docs.aux4.io/              # Documentation site
├── config-github-actions/     # CI/CD for config package
├── pkger-github-actions/      # CI/CD for pkger
├── claude-skills/             # Claude Code skills for aux4
└── ...
```

## Key Conventions

- The `main` profile is the required entry point in every `.aux4` file
- Use `profile:name` executor to create subcommand groups
- Nested profiles should have the same name as the command that routes to them; deeper nesting uses `<parent-profile>:<command>` (e.g., `email:list`, `ai:agent:config`)
- Command and profile composed names use dashes `-` (e.g., `send-email`, `my-command`)
- Only variable names use camelCase (e.g., `firstName`, `configFile`)
- Variables use `${varname}` syntax in execute arrays
- Man page filenames use `__` (double underscore) for hierarchy: `config__get.md`. If the profile or command name contains special characters (like `:`), replace them with a single `_` (e.g., profile `email:list` command `all` → `email_list__all.md`)
- Test files use `.test.md` extension with markdown-based execute/expect blocks. Same naming rule applies (e.g., `email_list__all.test.md`)
- Go packages cross-compile to 8 platform/arch combinations
- JS packages bundle with Rollup to a single `.mjs` file
- Publishing uses `aux4/publish-package-action@v1` GitHub Action
- Package registry: hub.aux4.io

## Skills Available

| Skill | When to Use |
|-------|-------------|
| `/aux4` | Understanding how aux4 works |
| `/aux4-package` | Creating a new aux4 package from scratch |
| `/aux4-command` | Adding commands to an existing `.aux4` file |
| `/aux4-test` | Writing or running `.test.md` tests |
| `/aux4-docs` | Updating README.md and man pages for an existing package |
| `/aux4-config` | Working with config.yaml files and the config package |
| `/aux4-agent` | Creating AI agents using aux4/ai-agent |
| `/aux4-copilot-skill` | Creating copilot skills with AI-driven or shell-based patterns |

## Important Rules

- **Run tests from `package/` or `package/test/`** — if you get "Command not found", you are in the wrong directory. Tests do NOT require installing the package locally — just build and run from the `package/` directory
- **Always build before testing** — run `npm run build` (JS) or `aux4 build` (Go) from the project root first
- **Test local packages with `aux4 aux4 releaser install`** — run from the `package/` directory to build, install locally, and test. Never manually update `~/.aux4.config/packages/`
- **Never browse or modify `~/.aux4.config/packages/` directly** — use `aux4 aux4 pkger list` to inspect installed packages, `aux4 aux4 pkger uninstall <scope>/<name>` to remove, and `aux4 aux4 releaser install` to install
- **Remove zip files from `package/` before building** — leftover `.zip` files get included in the package by mistake
- **If reinstalling a package fails** because the existing version is used by other packages, bump the version and install without uninstalling first
- **Never commit `.aux4` files with `-local` version suffix** — the `-local` suffix is added temporarily by `aux4 aux4 releaser install` for local testing; ensure it's not in the committed version
- **Tests must call `aux4 <command>`** — never call binaries or scripts directly (e.g., `node lib/tool.mjs` or `./dist/binary`). Tests should always exercise the full aux4 command path
- **Always format JSON with indentation** in `.aux4`, `.test.md`, `.md`, and any other files — never single-line compact JSON
- **Always specify a language tag** on fenced code blocks (`bash`, `json`, `yaml`, `text`, etc.) — never use bare ` ``` `
- **Use `file:<filename>` blocks** in tests to create fixture files — never use `cat <<EOF`, heredocs, or `echo >`
- **Use `expect`/`error` blocks** in tests to validate output — never suppress stderr with `2>/dev/null`
- **Use `beforeAll`/`afterAll` hooks** for daemon/server lifecycle in tests — never start/stop services inside `execute` blocks
- **Every command must have a man page** — do not skip any when adding new commands
- **Keep README in sync** — update it whenever commands are added or modified
- **Check security and vulnerabilities** — run `npm audit` (JS), review code for injection and common vulnerabilities
- **Check dependencies are up to date** — run `npm outdated` (JS) or `go list -m -u all` (Go)
- **Publish workflows must run tests** before publishing — add a test step in GitHub Actions

## Common Workflows

### Creating a new package
1. Use `/aux4-package` to scaffold the package
2. Use `/aux4-command` to add commands
3. Use `/aux4-test` to write tests
4. Build, then run tests from `package/`: `npm run build && cd package && aux4 test run`
5. Install locally to verify: `cd package && aux4 aux4 releaser install`
6. Use `aux4 aux4 releaser release` to publish

### Adding features to an existing package
1. Read the existing `.aux4` file
2. Use `/aux4-command` to add the new command
3. Use `/aux4-test` to add tests
4. Use `/aux4-docs` to update README.md and man pages
5. Build, then run tests from `package/`: `npm run build && cd package && aux4 test run`
6. Install locally to verify: `cd package && aux4 aux4 releaser install`

### Updating documentation
1. Use `/aux4-docs` to understand the doc structure and conventions
2. Update README.md and man pages

### Working with configuration
1. Use `/aux4-config` to understand config.yaml patterns
2. Add `aux4/config` as a dependency in `.aux4`
3. Use `--configFile` and `--config` flags in commands

### Creating an AI agent
1. Use `/aux4-package` to scaffold the package
2. Use `/aux4-agent` to set up the agent (instructions, config, commands)
3. Use `/aux4-test` to write tests
4. Build, then run tests from `package/`: `npm run build && cd package && aux4 test run`
5. Install locally to verify: `cd package && aux4 aux4 releaser install`
6. Use `aux4 aux4 releaser release` to publish

### Creating a copilot skill
1. Use `/aux4-copilot-skill` to scaffold the skill package and choose the pattern (AI-driven or shell-based)
2. Use `/aux4-test` to write tests
3. Use `/aux4-docs` to update README.md and man pages
4. Install locally to verify: `cd package && aux4 aux4 releaser install`
5. Verify with `aux4 copilot skills --help` and test end-to-end
6. Use `aux4 aux4 releaser release` to publish
