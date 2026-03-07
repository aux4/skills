---
name: aux4-package
description: Creates aux4 packages with .aux4 metadata, LICENSE, README, tests, man pages, and GitHub Actions for publishing to hub.aux4.io.
user-invocable: true
disable-model-invocation: false
argument-hint: [package-description]
---

# Create an aux4 Package

Create a complete aux4 package based on: $ARGUMENTS

**Before starting**, load all required skills by invoking `/aux4-test`, `/aux4-command`, and `/aux4-docs` so you have the full test format, command conventions, and documentation format available. Do not wait until you need them — load them now.

## Package Structure

Every aux4 package lives inside a `package/` directory with this structure:

```
package/
├── .aux4           # REQUIRED: Package metadata and command definitions
├── LICENSE         # REQUIRED: License file
├── README.md       # REQUIRED: Package documentation
├── man/            # Optional: Command manual pages
│   └── command__subcommand.md
├── test/           # Optional: Test files
│   └── command.test.md
├── lib/            # Optional: Scripts and supporting files
└── dist/           # Optional: Platform-specific binaries
    ├── darwin/
    │   ├── amd64/
    │   └── arm64/
    ├── linux/
    │   ├── amd64/
    │   ├── arm64/
    │   └── 386/
    └── windows/
        ├── amd64/
        ├── arm64/
        └── 386/
```

## Step 1: Create the .aux4 File

The `.aux4` file is JSON with package metadata and command profiles.

### Package Metadata Fields

```json
{
  "scope": "<org-or-username>",
  "name": "<package-name>",
  "version": "1.0.0",
  "description": "<short description>",
  "license": "MIT",
  "tags": ["tag1", "tag2"],
  "dependencies": ["aux4/config"],
  "system": [
    ["test:node --version", "brew:node", "apt:nodejs", "dnf:nodejs", "apk:nodejs"]
  ],
  "profiles": [...]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `scope` | Yes | Package namespace (org or username) |
| `name` | Yes | Package name |
| `version` | Yes | Semantic version (e.g., `1.0.0`) |
| `description` | Yes | Short description |
| `license` | Yes | License identifier (MIT, Apache-2.0, etc.) |
| `tags` | No | Searchable tags |
| `dependencies` | No | Other aux4 packages needed (`"scope/name"` or `"scope/name@version"`) |
| `system` | No | System dependencies (see below) |
| `profiles` | Yes | Array of command profiles |

### System Dependencies Format

When a package requires external tools (CLI binaries, libraries, or runtimes), declare them in `system`. Each entry is an array where:

- The first element is a `test:` command that checks if the tool is already installed
- The remaining elements are `<prefix>:<package>` pairs for installing it with different package managers

```json
"system": [
  ["test:node --version", "brew:node", "apt:nodejs", "linux:nodejs"],
  ["test:jq --version", "brew:jq", "apt:jq", "dnf:jq"],
  ["test:prettier --version", "npm:prettier"]
]
```

When the package is installed via pkger, aux4 runs the `test:` command first. If it fails (tool not found), it tries to install using one of the available system installers: `aux4 aux4 pkger system <prefix> install <package>`.

| Prefix | Package Manager | Platform |
|--------|----------------|----------|
| `brew` | Homebrew | macOS |
| `apt` | APT | Debian/Ubuntu |
| `dnf` | DNF | Fedora/RHEL |
| `yum` | YUM | CentOS/RHEL |
| `apk` | APK | Alpine |
| `npm` | npm | Cross-platform (Node.js) |
| `pkgx` | pkgx | Cross-platform |
| `linux` | Alias for `apt` + `dnf` + `yum` + `apk` | All Linux |

Common examples:

```json
"system": [
  ["test:node --version", "brew:node", "linux:nodejs"],
  ["test:go version", "brew:go", "linux:golang"],
  ["test:python3 --version", "brew:python3", "apt:python3"],
  ["test:prettier --version", "npm:prettier"],
  ["test:jq --version", "brew:jq", "apt:jq", "apk:jq"]
]
```

### Profiles and Commands

The `main` profile is the entry point. Use `profile:` executor for subcommand groups. The profile name should match the command name that routes to it. For deeper nesting, use `<parent-profile>:<command>` as the profile name (e.g., `email:list`, `ai:agent:config`):

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "mytool",
          "execute": ["profile:mytool"],
          "help": { "text": "My tool commands" }
        }
      ]
    },
    {
      "name": "mytool",
      "commands": [
        {
          "name": "run",
          "execute": ["echo \"Running ${file}\""],
          "help": {
            "text": "Run a file",
            "variables": [
              {
                "name": "file",
                "text": "File to run",
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

### Naming, Variables, and Executors

- **Command/profile names**: dashes for composed names (`send-email`, `run-all`)
- **Variable names**: camelCase (`firstName`, `configFile`)

See `/aux4` for the full reference on variable definitions (properties, dot notation, resolution order), executor prefixes (`profile:`, `set:`, `log:`, `nout:`, `json:`, `each:`, `confirm:`, `stdin:`, `alias:`, `#`), and parameter functions (`value()`, `values()`, `param()`, `params()`, `object()`).

## Step 2: Create the LICENSE File

Use a standard license text (MIT, Apache-2.0, etc.). You can also generate it with:

```bash
aux4 license use --name mit --owner "Owner" --year 2025 --project "project-name"
```

## Step 3: Create the README.md

Include: description, installation instructions, usage examples, and command reference. See `/aux4-docs` for the full README structure, section order, and formatting conventions.

## Step 4: Create Man Pages

Place markdown files in `package/man/`. Name them using double underscores for command hierarchy:

| Command | Man Page Filename |
|---------|-------------------|
| `aux4 mytool` | `mytool.md` |
| `aux4 mytool run` | `mytool__run.md` |
| `aux4 mytool run all` | `mytool__run__all.md` |
| `aux4 email list all` | `email_list__all.md` |

**Special characters in names**: If a profile or command name contains special characters (like `:`), replace them with a single `_` in the filename. For example, profile `email:list` with command `all` becomes `email_list__all.md`.

Man pages provide detailed documentation for each command, displayed via `aux4 aux4 man <command>`. Use `####` headings for Description, Usage, and Example:

````markdown
#### Description

The `run` command processes input files and outputs the result. It supports multiple formats and automatically detects the input encoding. If no output file is specified, the result is written to stdout.

#### Usage

```bash
aux4 mytool run --format <json|csv|yaml> [--output <file>]
```

--format  The output format to use (required)
--output  Path to write the output file (default: stdout)

#### Example

```bash
aux4 mytool run --format csv --output result.csv
```

```text
Processing complete. Output written to result.csv
```
````

The description should go beyond the short help text — explain what the command does, when to use it, how flags interact, and any important behaviors. Include realistic examples with expected output. Create one man page per command. See `/aux4-docs` for the full documentation conventions.

## Step 5: Create Tests

Place `.test.md` files in `package/test/`. Name files using double underscores for command hierarchy (e.g., `mytool__run.test.md` for `aux4 mytool run`), matching the man page convention. If the profile or command name contains special characters (like `:`), replace them with a single `_` (e.g., `email_list__all.test.md` for profile `email:list` command `all`). Test files are published to hub.aux4.io as usage examples, so they also serve as documentation. See the `/aux4-test` skill for the full test format.

**Important**: Tests in `package/test/` should call `aux4 <command>` directly — do NOT create `file:.aux4` blocks to redefine the package commands. The test runner automatically discovers `package/.aux4` from the parent directory. Only use `file:.aux4` for standalone command tests that are not part of a package.

Quick example for a package test:

````markdown
# mytool run

## should output the filename

```execute
aux4 mytool run myfile.txt
```

```expect
Running myfile.txt
```
````

## Step 6: Set Up Build, CI/CD, and Publishing

Read `../references/build-configuration.md` for the complete build setup, including:
- Go cross-compilation (8 platform/arch targets, builder profiles)
- JavaScript Rollup bundling (single `.mjs` output)
- JS packages with native dependencies
- Referencing files in commands (`${packageDir}/binary` vs `node ${packageDir}/lib/bundle.mjs`)
- Key differences between Go and JavaScript packages
- `.gitignore` templates for Go, JS, and Shell-only packages
- GitHub Actions publish workflow (`aux4/publish-package-action@v1`)
- Symlink for local Go testing
- Build, install locally, and publish commands (pkger + package-releaser)

## Instructions

When creating a package, always:

1. Ask the user for scope, name, description, and license if not provided in the arguments.
2. Ask if it's a Go package (multi-platform binaries) or JavaScript package (Node.js bundle) or a simple shell-only package.
3. Create the `package/` directory with `.aux4`, `LICENSE`, and `README.md`.
4. Design a clean command hierarchy using profiles for subcommand groups.
5. Add man pages in `package/man/` for each command.
6. Add tests in `package/test/` using `.test.md` format. Tests should call `aux4 <command>` directly — do NOT use `file:.aux4` blocks. The test runner auto-discovers `package/.aux4` from the parent directory.
7. Create the root `.aux4` with the appropriate build configuration:
   - Go: builder profiles for darwin/linux/windows with `go build` commands (use `cd ${packageDir} &&` prefix)
   - JS: simple `npm run build` + `rollup.config.js` + `package.json`
   - Shell-only: empty build or no build needed
8. Create a `.gitignore` file excluding build artifacts (`package/dist/`, `node_modules/`, etc.), platform files (`.DS_Store`), and language-specific generated files (`go.sum`, symlinks).
9. Create `.github/workflows/publish.yml` with the appropriate build steps for the package type.
10. Use `${packageDir}` to reference files within the package.
11. Use `value()`, `values()`, `param()`, `params()` for parameter formatting in execute commands.
12. For Go packages, commands reference `${packageDir}/binary-name`.
13. For JS packages, commands reference `node ${packageDir}/lib/bundle.mjs` and include Node.js in system dependencies.
14. When writing markdown files (man pages, `.test.md`, `README.md`), use 4 backticks (````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
15. For Go packages, before running tests: build the binary (`aux4 build`), then create a symlink from `package/<binary-name>` to `dist/<os>/<arch>/<binary-name>` for the current platform.
16. When any command is added or modified, always update the corresponding man page, test file, and README.md to stay in sync. See `/aux4-docs` for documentation conventions.
17. Load all needed skills at the start of package creation (e.g., `/aux4-package`, `/aux4-test`, `/aux4-command`, `/aux4-docs`) rather than loading them one by one as needed.
