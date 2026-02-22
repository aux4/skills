---
name: aux4
description: Explains how aux4 works - CLI generator, .aux4 files, profiles, commands, variables, executors, and the packaging ecosystem.
user-invocable: true
disable-model-invocation: false
---

# aux4 - CLI Generator

aux4 is a CLI (Command-Line Interface) generator that creates high-level scripts to automate daily tasks. It uses JSON-based `.aux4` configuration files to define command hierarchies.

## Core Concepts

### The .aux4 File

A `.aux4` file is a hidden JSON file that defines profiles and commands. aux4 searches for `.aux4` files in the current directory and parent directories.

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "hello",
          "execute": [
            "echo \"Hello, ${name}!\""
          ],
          "help": {
            "text": "Say hello",
            "variables": [
              {
                "name": "name",
                "text": "Name to greet",
                "default": "World"
              }
            ]
          }
        }
      ]
    }
  ]
}
```

Usage: `aux4 hello --name John` outputs `Hello, John!`

### Profiles

Profiles group related commands. The `main` profile is required and is the default entry point.

Commands can switch to another profile using the `profile:` executor:

```json
{
  "name": "deploy",
  "execute": ["profile:deploy"],
  "help": { "text": "Deployment commands" }
}
```

This means `aux4 deploy <subcommand>` will look for commands in the `deploy` profile.

### Variables

Variables are parameters for commands. They support:

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Variable name, referenced as `${name}` in execute |
| `text` | string | Description shown in help and prompts |
| `default` | string | Default value (skips prompt if set) |
| `arg` | boolean | Accept as positional argument (only one variable per command can be `arg: true`) |
| `multiple` | boolean | Accept multiple values |
| `env` | string | Read from environment variable |
| `options` | string[] | List of choices for select prompt |
| `hide` | boolean | Hide input (for passwords) |
| `encrypt` | boolean | Encrypt the value |

**Resolution order**: argument > environment variable > config > encrypted > default > interactive prompt

**Special variables**:
- `${response}` - Output of previous command
- `${packageDir}` - Directory of the `.aux4` file
- `${aux4HomeDir}` - aux4 home directory (`~/.aux4.config`)

### Executors (Special Prefixes)

Commands in the `execute` array can use special prefixes:

| Prefix | Example | Purpose |
|--------|---------|---------|
| *(none)* | `echo "hi"` | Run shell command |
| `profile:` | `profile:deploy` | Switch to another profile |
| `set:` | `set:url=https://api.com` | Set a variable |
| `log:` | `log:Processing ${file}` | Print output |
| `debug:` | `debug:value is ${x}` | Debug output (AUX4_DEBUG=true) |
| `nout:` | `nout:curl -s ${url}` | Run without output, saves to `${response}` |
| `json:` | `json:${response}` | Parse JSON output |
| `each:` | `each:${response}` | Iterate over lines/array |
| `confirm:` | `confirm:Are you sure?` | Yes/no prompt |
| `stdin:` | `stdin:command` | Pass stdin to command |
| `alias:` | `alias:command` | Share stdio with parent |
| `#` | `# this is a comment` | Comment (no-op) |

### Functions (Parameter Formatting)

Used in `execute` strings to format variables:

| Function | Example | Output |
|----------|---------|--------|
| `value(name)` | `command value(file)` | `command myfile.txt` |
| `values(a, b)` | `command values(host, port)` | `command 'localhost' '3000'` |
| `param(name)` | `command param(file)` | `command --file 'myfile.txt'` |
| `params(a, b)` | `command params(host, port)` | `command --host 'localhost' --port '3000'` |
| `object(a, b)` | `command object(host, port)` | `command '{"host":"localhost","port":3000}'` |
| `if(a == b)` | `if(env == prod)` | Conditional execution |

**Nested field access**: Use dot notation to access nested config/variable values. For example, with `person: { firstName: John, lastName: Doe }` in config:
- `values(person.firstName, person.lastName)` resolves to `'John' 'Doe'`
- `values(person, person.firstName, person.lastName)` resolves to `'{"firstName":"John","lastName":"Doe"}' 'John' 'Doe'` — passing the parent object as JSON and the individual fields as separate arguments

This works with `value()`, `values()`, `param()`, `params()`, and `object()`. The external command (JS, Go, etc.) receives these as fixed positional arguments at known indices — it does not need to implement config parsing or key lookup.

### Configuration Files (config.yaml)

Requires `aux4/config` package. Use `--configFile config.yaml --config` flags.

```yaml
config:
  dev:
    host: localhost
    port: 3000
```

The `--config` flag automatically extracts values from the config file and populates command variables. You do **not** need to call `aux4 config get` to retrieve individual values — aux4 handles it automatically.

Access nested values with `/`: `aux4 config get dev/host` returns `localhost`. The `config get` command is only needed for programmatic access outside of normal command variable resolution.

### Package Ecosystem

- **hub.aux4.io** - Package registry
- **pkger** - Package manager (`aux4 aux4 pkger install scope/name`)
- **Packages** are ZIP files containing `.aux4`, `LICENSE`, `README.md`, and optional `lib/`, `dist/`, `man/`, `test/` directories
- **Dependencies** - aux4 packages (`"dependencies": ["aux4/config"]`) and system packages (`"system": [["test:node --version", "brew:node"]]`)

### Installation

```bash
curl https://aux4.sh | sh     # Easy install
brew install aux4              # Homebrew
npm install -global aux4       # npm
```

### Investigating Commands

Use `--help` on any command to see its subcommands, variables, and descriptions:

```bash
aux4 --help                    # List all top-level commands
aux4 deploy --help             # List subcommands under deploy
aux4 deploy run --help         # Show variables for deploy run
```

Use `--showSource` to see the execute instructions for a command:

```bash
aux4 deploy run --showSource   # Show the execute array for deploy run
```

### Built-in Commands

```bash
aux4 aux4 version              # Show version
aux4 aux4 man <command>        # Show command man page
aux4 aux4 source <command>     # Show command source (.aux4 definition)
aux4 aux4 which <command>      # Show where command is defined
```

### Package Manager (pkger)

```bash
aux4 aux4 pkger list           # List installed packages
aux4 aux4 pkger man <package>  # Show package documentation (README.md)
aux4 aux4 pkger install <pkg>  # Install a package from hub.aux4.io
aux4 aux4 pkger uninstall <pkg> # Uninstall a package
```

Examples:

```bash
aux4 aux4 pkger list                     # See all installed packages
aux4 aux4 pkger man aux4/config          # Read the config package docs
aux4 aux4 pkger install aux4/db-sqlite   # Install db-sqlite package
```
