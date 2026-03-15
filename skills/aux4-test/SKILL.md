---
name: aux4-test
description: Creates and runs aux4 .test.md files for testing aux4 packages. Markdown-based test format with execute, expect, error, file blocks, and hooks.
user-invocable: true
disable-model-invocation: false
argument-hint: [test-description]
---

# aux4 Test Framework

Create or run aux4 tests based on: $ARGUMENTS

## Overview

aux4 uses markdown-based `.test.md` files for testing. Tests are placed in the `package/test/` directory. The test runner parses markdown headings and fenced code blocks to build and execute test suites.

## Running Tests

**IMPORTANT:** You must run `aux4 test run` from the `package/` or `package/test/` directory. If you get "Command not found", you are in the wrong directory — the test runner discovers `package/.aux4` from the current or parent directory. You do NOT need to install the package locally to run tests — just build and run from the `package/` directory.

**Always build before testing.** Run `npm run build` (JS) or `aux4 build` (Go) from the project root before running tests to ensure the bundle/binary is up to date.

```bash
# Build first (from project root)
npm run build        # JS packages
aux4 build           # Go packages

# Run all tests (from package/ or package/test/)
cd package && aux4 test run

# Run a specific test file
aux4 test run package/test/mycommand.test.md

# Add a test programmatically
aux4 test add mytest.test.md --level 2 --name "Test Name" --execute "echo hello"
```

## Test File Format

Test files use markdown with special fenced code blocks. The heading hierarchy (`#`, `##`, `###`, etc.) defines test grouping (like `describe` blocks).

### Basic Test Structure

````markdown
# Command Name

## Scenario Description

### should do something

```execute
aux4 mycommand --flag value
```

```expect
expected output
```
````

### Complete Example

````markdown
# greet hello

## with default name

### should greet World

```execute
aux4 greet hello
```

```expect
Hello, World!
```

## with custom name

### should greet the given name

```execute
aux4 greet hello --name John
```

```expect
Hello, John!
```
````

## Code Block Types

### `execute` - Run a Command

The shell command to execute. Captures stdout and stderr separately.

````markdown
```execute
aux4 config get dev/host
```
````

### `expect` - Validate stdout

Exact match against stdout by default. Must follow an `execute` block.

````markdown
```expect
localhost
```
````

### `expect` with Modifiers

#### `:partial` - Substring/Wildcard Matching

Matches a substring of the output. Supports wildcards:
- `*?` - matches any single line content
- `*` - matches any content on the same line
- `**` - matches any content across multiple lines

````markdown
```expect:partial
Hello *?
```
````

````markdown
```expect:partial
Start ** end
```
````

#### `:ignoreCase` - Case-Insensitive

````markdown
```expect:ignoreCase
hello world
```
````

#### `:regex` - Regular Expression

````markdown
```expect:regex
Hello, \w+!
```
````

#### `:json` - JSON Matching

Parses the command output as JSON and compares it with the expected JSON. This lets you write formatted JSON in the expect block even if the command outputs compact JSON on a single line:

````markdown
```execute
aux4 config get dev
```

```expect:json
{
  "host": "localhost",
  "port": 3000
}
```
````

Can be combined with other modifiers like `:partial`:

````markdown
```expect:json:partial
{
  "host": "localhost"
}
```
````

#### Combined Modifiers

Modifiers can be combined with `:`:

````markdown
```expect:regex:ignoreCase
error code: [a-z]+\d+
```
````

````markdown
```expect:partial:ignoreCase
hello *?
```
````

### `error` - Validate stderr

Same syntax and modifiers as `expect`, but matches against stderr:

````markdown
```error
Error: file not found
```
````

````markdown
```error:partial
Error: *?
```
````

### Combined stdout and stderr

You can validate both stdout and stderr for the same execute:

````markdown
```execute
echo "Success" && echo "Warning" >&2
```

```expect
Success
```

```error
Warning
```
````

### `file:<filename>` - Create Test Fixtures

Creates a file before the test runs. Scoped to the current heading and nested headings.

````markdown
```file:config.yaml
config:
  dev:
    host: localhost
    port: 3000
```
````

````markdown
```file:.aux4
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "hello",
          "execute": ["echo \"Hello, ${name}!\""],
          "help": {
            "text": "Say hello",
            "variables": [
              { "name": "name", "text": "Name", "default": "World" }
            ]
          }
        }
      ]
    }
  ]
}
```
````

**Scoping rules:**
- Files declared at a parent heading are available to all nested tests
- Files are NOT shared between sibling headings
- Files are automatically cleaned up after tests

### `timeout` - Set Command Timeout

Override the default timeout (in milliseconds). Place before the `execute` block:

````markdown
```timeout
5000
```

```execute
sleep 2 && echo "Done"
```

```expect
Done
```
````

### `beforeAll` / `afterAll` - Scenario Hooks

Run once before/after all tests in the current heading scope:

````markdown
## My Scenario

```beforeAll
mkdir -p test-dir
echo "ready" > test-dir/setup.log
```

```afterAll
rm -rf test-dir
```

### test 1

```execute
cat test-dir/setup.log
```

```expect
ready
```
````

### `beforeEach` / `afterEach` - Per-Test Hooks

Run before/after each individual test in the current heading scope:

````markdown
## My Scenario

```beforeEach
echo "fresh" > state.txt
```

```afterEach
rm -f state.txt
```

### test 1

```execute
cat state.txt
```

```expect
fresh
```
````

## Writing Tests for aux4 Packages

### Package Tests (in `package/test/`)

When tests live inside `package/test/`, the test runner automatically discovers the `package/.aux4` file from the parent directory. **Do NOT create `file:.aux4` blocks** — just call `aux4 <command>` directly:

````markdown
# greet hello

## with default name

### should greet World

```execute
aux4 greet hello
```

```expect
Hello, World!
```

## with custom name

### should greet the given name

```execute
aux4 greet hello --name Alice
```

```expect
Hello, Alice!
```
````

### Standalone Tests (outside a package)

Only use `file:.aux4` blocks when testing commands that are NOT part of a package (e.g., standalone test files or testing inline command definitions):

````markdown
# my standalone test

```file:.aux4
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "hello",
          "execute": ["echo \"Hello, ${name}!\""],
          "help": {
            "text": "Say hello",
            "variables": [
              { "name": "name", "text": "Name", "default": "World" }
            ]
          }
        }
      ]
    }
  ]
}
```

## should greet World

```execute
aux4 hello
```

```expect
Hello, World!
```
````

### Testing Config Files

````markdown
# config get

```file:config.yaml
config:
  dev:
    host: localhost
    port: 3000
```

## get nested value

```execute
aux4 config get dev/host
```

```expect
localhost
```
````

### Testing Error Cases

````markdown
# error handling

## invalid input

```execute
aux4 config get --file nonexistent.yaml dev
```

```error:partial
Error *?
```
````

### Testing JSON Output

Use `expect:json` to compare JSON output — it parses both the command output and expected value as JSON, so formatting differences don't matter:

````markdown
# json output

```execute
aux4 config get dev
```

```expect:json
{
  "host": "localhost",
  "port": 3000
}
```
````

### Testing Long-Running Processes (Servers, Daemons)

For tests that require a background server or daemon, **always use `beforeAll`/`afterAll` hooks** to manage the lifecycle. Never start/stop servers inside `execute` blocks or use inline `> /dev/null 2>&1 &` hacks in test commands.

````markdown
## daemon tests

```beforeAll
nohup aux4 myservice start >/dev/null 2>&1 &
sleep 2
```

```afterAll
aux4 myservice stop
```

### should respond to requests

```execute
aux4 myservice status
```

```expect
running
```
````

Read `../references/build-configuration.md` for the full pattern with examples.

### Preparing Go Packages for Testing

For Go packages, you must build and create a symlink before running tests. See `../references/build-configuration.md` for the build + symlink commands.

## Test File Naming Conventions

- Place in `package/test/` directory
- Use double underscores (`__`) to separate command hierarchy levels in the filename, matching the man page convention:

| Command | Test File |
|---------|-----------|
| `aux4 greet` | `greet.test.md` |
| `aux4 greet hello` | `greet__hello.test.md` |
| `aux4 config get` | `config__get.test.md` |
| `aux4 pdf parse` | `pdf__parse.test.md` |
| `aux4 repository read` | `repository__read.test.md` |
| `aux4 email list all` | `email_list__all.test.md` |

**Special characters in names**: If a profile or command name contains special characters (like `:`), replace them with a single `_` in the filename. For example, profile `email:list` with command `all` becomes `email_list__all.test.md`.

- Test files are published to hub.aux4.io as usage examples alongside man pages, so they also serve as documentation
- Use descriptive heading hierarchy:
  - `#` - Command or feature name
  - `##` - Scenario or context
  - `###` - Individual test case (prefixed with "should" for clarity)

## Best Practices

1. Start each test file with a `#` heading describing the command being tested
2. Use `file:` blocks at the appropriate scope level for test fixtures — **never use `cat <<EOF` or heredocs** in `execute` or `beforeAll` blocks to create files; always use `file:<filename>` blocks instead
3. Prefer `expect:partial` for output that may vary (timestamps, paths)
4. Use `expect:regex` for pattern matching complex output
5. Keep tests focused - one `execute` block per test case
6. Clean up side effects with `afterAll` or `afterEach` hooks
7. Test both success and error cases — use `error` or `error:partial` blocks for expected stderr output instead of suppressing with `2>/dev/null`
8. **Never suppress output with `> /dev/null 2>&1`** in `execute` blocks — use `expect` for stdout and `error` for stderr. Suppressing output hides real errors and makes debugging impossible
9. Use meaningful heading names that describe the behavior being tested
10. **Test file names must match what they test** — if a test checks hub.aux4.io, don't name it `google-search.test.md`
11. **Always specify a language tag on fenced code blocks** — use `bash`, `json`, `yaml`, `text`, etc. Never use bare ` ``` ` without a language

## Instructions

When creating tests:

1. Determine what commands need testing based on the `.aux4` file.
2. Create a `.test.md` file for each major command or feature. **Name files after what they actually test.**
3. For package tests (in `package/test/`), call `aux4 <command>` directly — do NOT use `file:.aux4` blocks. The test runner auto-discovers `package/.aux4` from the parent directory. Only use `file:.aux4` for standalone tests outside a package.
4. Write tests for all command variations (default values, flags, args, errors).
5. Use appropriate expect modifiers for flexible matching.
6. Group related tests under descriptive headings.
7. For tests that need a background server or daemon, use `nohup ... >/dev/null 2>&1 &` in `beforeAll` and clean up in `afterAll`. Never start/stop daemons inside `execute` blocks.
8. For Go packages, ensure the binary is built and a symlink exists at `package/<binary-name>` before running tests.
9. When writing `.test.md` files, use 4 backticks (````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
10. **Always format JSON with indentation** in `file:` blocks (`.aux4`, `config.yaml`, fixture files). Never use single-line compact JSON for `.aux4` definitions or config fixtures. Each item in the `execute` array must be on its own line.
11. **Use `file:<filename>` blocks** to create test fixture files — never use `cat <<EOF`, heredocs, or `echo >` in `execute` or `beforeAll` blocks to create files.
12. **Use `expect` and `error` blocks** to validate output — never suppress stderr with `2>/dev/null`. Use `error:partial` for expected error messages.
13. **Always specify a language tag** on fenced code blocks (`bash`, `json`, `yaml`, `text`, etc.). Never use bare ` ``` ` without a language.
14. **Always build before running tests.** Run `npm run build` (JS) or `aux4 build` (Go) from the project root first.
15. **Run tests from `package/` or `package/test/`** — if you get "Command not found", you are in the wrong directory. You do NOT need to install the package locally — just build and run.
16. **Tests must always call `aux4 <command>`** in `execute` blocks — never call binaries or scripts directly (e.g., `node lib/tool.mjs`, `./dist/binary`). Always test through the full aux4 command path.
