# Build Configuration Reference

This reference covers build setup, CI/CD, and publishing for aux4 packages.

## Go Package (Multi-Platform Binaries)

Go packages compile to native binaries for each platform. The root `.aux4` defines cross-compilation:

```
my-go-package/
├── .aux4              # Build configuration (builder profiles)
├── go.mod
├── go.sum
├── main.go            # Go source code
└── package/
    ├── .aux4          # Package metadata (commands use ${packageDir}/my-binary)
    ├── LICENSE
    ├── README.md
    ├── man/
    ├── test/
    └── dist/          # Compiled binaries per platform
        ├── darwin/amd64/my-binary
        ├── darwin/arm64/my-binary
        ├── linux/amd64/my-binary
        ├── linux/arm64/my-binary
        ├── linux/386/my-binary
        ├── windows/amd64/my-binary.exe
        ├── windows/arm64/my-binary.exe
        └── windows/386/my-binary.exe
```

Root `.aux4` for Go packages (note: `cd ${packageDir} &&` ensures builds work from any directory):

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "build",
          "execute": ["aux4 builder all"],
          "help": { "text": "Build for all platforms" }
        },
        {
          "name": "builder",
          "execute": ["profile:builder"],
          "help": { "text": "Platform-specific builds" }
        }
      ]
    },
    {
      "name": "builder",
      "commands": [
        {
          "name": "darwin",
          "execute": [
            "cd ${packageDir} && GOOS=darwin GOARCH=amd64 go build -o package/dist/darwin/amd64/my-binary .",
            "cd ${packageDir} && GOOS=darwin GOARCH=arm64 go build -o package/dist/darwin/arm64/my-binary ."
          ],
          "help": { "text": "Build for macOS" }
        },
        {
          "name": "linux",
          "execute": [
            "cd ${packageDir} && GOOS=linux GOARCH=amd64 go build -o package/dist/linux/amd64/my-binary .",
            "cd ${packageDir} && GOOS=linux GOARCH=arm64 go build -o package/dist/linux/arm64/my-binary .",
            "cd ${packageDir} && GOOS=linux GOARCH=386 go build -o package/dist/linux/386/my-binary ."
          ],
          "help": { "text": "Build for Linux" }
        },
        {
          "name": "windows",
          "execute": [
            "cd ${packageDir} && GOOS=windows GOARCH=amd64 go build -o package/dist/windows/amd64/my-binary.exe .",
            "cd ${packageDir} && GOOS=windows GOARCH=arm64 go build -o package/dist/windows/arm64/my-binary.exe .",
            "cd ${packageDir} && GOOS=windows GOARCH=386 go build -o package/dist/windows/386/my-binary.exe ."
          ],
          "help": { "text": "Build for Windows" }
        },
        {
          "name": "all",
          "execute": [
            "aux4 builder darwin",
            "aux4 builder linux",
            "aux4 builder windows",
            "chmod -R +x package/dist/*"
          ],
          "help": { "text": "Build for all platforms" }
        }
      ]
    }
  ]
}
```

`${packageDir}` always points to the directory where the `package/.aux4` file lives (the `package/` folder), so `${packageDir}/dist/...` resolves to `package/dist/...`. Commands reference the binary directly:

```json
"execute": ["${packageDir}/my-binary value(arg1)"]
```

Or with stdin:

```json
"execute": ["stdin:${packageDir}/my-binary command values(a, b)"]
```

## JavaScript Package (Rollup Bundle)

JS packages bundle source + dependencies into a single file using Rollup. They require Node.js at runtime and declare it as a system dependency. The bundled output goes to `package/lib/`.

`${packageDir}` always points to the directory where the `package/.aux4` file lives (the `package/` folder), so `${packageDir}/lib/my-tool.mjs` resolves to `package/lib/my-tool.mjs`.

```
my-js-package/
├── .aux4              # Build configuration (npm run build)
├── package.json       # NPM dependencies and build scripts
├── rollup.config.js   # Rollup bundler configuration
├── bin/
│   └── executable.js  # Entry point (#!/usr/bin/env node)
├── lib/               # Source code
│   └── ...
└── package/
    ├── .aux4          # Package metadata (commands use node ${packageDir}/lib/...)
    ├── LICENSE
    ├── README.md
    ├── man/
    ├── test/
    └── lib/
        └── my-tool.mjs  # Bundled output (single file)
```

Root `.aux4` for JS packages:

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "build",
          "execute": ["npm run build"],
          "help": { "text": "Build the package" }
        }
      ]
    }
  ]
}
```

`package.json` with Rollup build:

```json
{
  "name": "@scope/my-tool",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "rollup -c"
  },
  "dependencies": {
    "colors": "^1.4.0"
  },
  "devDependencies": {
    "@rollup/plugin-commonjs": "^28.0.6",
    "@rollup/plugin-json": "^6.1.0",
    "@rollup/plugin-node-resolve": "^16.0.1",
    "rollup": "^4.46.3"
  }
}
```

`rollup.config.js` (ES Module output):

```javascript
import { nodeResolve } from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import json from '@rollup/plugin-json';

export default {
  input: 'bin/executable.js',
  output: {
    file: 'package/lib/my-tool.mjs',
    format: 'es',
    inlineDynamicImports: true
  },
  plugins: [
    nodeResolve({ preferBuiltins: true }),
    commonjs(),
    json()
  ],
  external: ['fs', 'path', 'stream', 'util', 'events', 'buffer', 'string_decoder', 'crypto', 'os', 'tty', 'process']
};
```

## JS Package with Native Dependencies

Some dependencies have native binaries (e.g., SQLite, libsql) that Rollup cannot bundle. In this case:

1. Exclude those dependencies from Rollup's `external` array
2. Add a `package.json` inside `package/lib/` with only those excluded dependencies
3. Update the build script to run `npm install` in `package/lib/` after bundling

```
my-js-package/
├── .aux4
├── package.json       # "build": "rollup -c && cd package/lib && npm install"
├── rollup.config.js   # Excludes native deps
└── package/
    ├── .aux4
    └── lib/
        ├── my-tool.js     # Bundled output
        └── package.json   # Only the excluded native dependencies
```

`rollup.config.js` — exclude native deps:

```javascript
export default {
  input: 'main.js',
  output: {
    file: 'package/lib/my-tool.js',
    format: 'es'
  },
  plugins: [
    nodeResolve({ preferBuiltins: true }),
    commonjs(),
    json()
  ],
  external: [
    'fs', 'path', 'crypto', 'util', 'stream', 'os', 'process',
    'libsql',           // native dependency — cannot be bundled
    /^@libsql\/.*/      // platform-specific binaries
  ]
};
```

`package.json` (root) — build script runs `npm install` in `package/lib/`:

```json
{
  "scripts": {
    "build": "rollup -c && cd package/lib && npm install"
  }
}
```

`package/lib/package.json` — only the excluded native dependencies:

```json
{
  "type": "module",
  "dependencies": {
    "libsql": "^0.5.20"
  }
}
```

Root `.aux4` stays the same (`aux4 build` runs `npm run build`), which triggers Rollup and then installs the native deps in `package/lib/`.

## Referencing Files in Commands

In `package/.aux4`, commands reference files relative to `${packageDir}` (the `package/` directory):

```json
{
  "system": [
    ["test:node --version", "brew:node", "apt:nodejs", "linux:nodejs"]
  ],
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "mytool",
          "execute": ["stdin:node ${packageDir}/lib/my-tool.mjs action values(a, b)"],
          "help": { "text": "Run my tool" }
        }
      ]
    }
  ]
}
```

## Key Differences: Go vs JavaScript

| Aspect | Go Package | JavaScript Package |
|--------|-----------|-------------------|
| Runtime | None (standalone binary) | Node.js required |
| Build | `GOOS=... go build` per platform | `rollup -c` (single bundle) |
| Output | `package/dist/{os}/{arch}/binary` | `package/lib/tool.mjs` |
| Execute | `${packageDir}/binary args` | `node ${packageDir}/lib/tool.mjs args` |
| Root .aux4 | Builder profiles (darwin/linux/windows/all) | Simple `npm run build` |
| System deps | Usually none | `["test:node --version", "brew:node", ...]` |
| Distribution | Platform-specific zips + master zip | Single zip (cross-platform) |

## .gitignore Templates

### Go Package .gitignore

```
package/dist/
package/<binary-name>
<binary-name>
go.sum
.DS_Store
```

The `package/<binary-name>` entry excludes the symlink used for local testing (see Symlink section below). The root `<binary-name>` excludes any binary built in the project root during development.

### JavaScript Package .gitignore

```
node_modules/
package/lib/*.mjs
package/lib/node_modules/
.DS_Store
```

### Shell-Only Package .gitignore

```
.DS_Store
```

## GitHub Actions

Create `.github/workflows/publish.yml` to auto-publish to hub.aux4.io:

```yaml
name: Publish Package

on:
  push:
    branches:
      - main
    paths:
      - '**'
      - '!.github/**'
      - '!README.md'
      - '!LICENSE'
      - '!.gitignore'

  workflow_dispatch:
    inputs:
      level:
        description: 'Release level (patch, minor, major)'
        required: true
        default: 'patch'
        type: choice
        options:
          - patch
          - minor
          - major

concurrency:
  group: publish-package
  cancel-in-progress: false

permissions:
  contents: write

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # Add build steps here if needed (see below)

      - name: Publish package
        uses: aux4/publish-package-action@v1
        with:
          dir: package
          level: ${{ inputs.level || 'patch' }}
          aux4_token: ${{ secrets.AUX4_ACCESS_TOKEN }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

**Required secrets:**
- `AUX4_ACCESS_TOKEN` - Authentication token for hub.aux4.io
- `GITHUB_TOKEN` - Provided automatically by GitHub

**Important**: The `dir: package` parameter tells the publish action to run from the `package/` directory where the `.aux4` metadata file lives.

For **Go** packages, add a build step before publish:

```yaml
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'

      - name: Build
        run: |
          GOOS=darwin GOARCH=amd64 go build -o package/dist/darwin/amd64/my-binary .
          GOOS=darwin GOARCH=arm64 go build -o package/dist/darwin/arm64/my-binary .
          GOOS=linux GOARCH=amd64 go build -o package/dist/linux/amd64/my-binary .
          GOOS=linux GOARCH=arm64 go build -o package/dist/linux/arm64/my-binary .
          GOOS=linux GOARCH=386 go build -o package/dist/linux/386/my-binary .
          GOOS=windows GOARCH=amd64 go build -o package/dist/windows/amd64/my-binary.exe .
          GOOS=windows GOARCH=arm64 go build -o package/dist/windows/arm64/my-binary.exe .
          GOOS=windows GOARCH=386 go build -o package/dist/windows/386/my-binary.exe .
```

For **JavaScript** packages, add build and test steps before publish:

```yaml
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run tests
        run: cd package && aux4 test run
```

For **Go** packages, add a test step after the build step:

```yaml
      - name: Run tests
        run: |
          ln -sf dist/linux/amd64/my-binary package/my-binary
          cd package && aux4 test run
```

**Always run tests before publishing.** The workflow should fail if tests don't pass, preventing broken packages from being published.

## Symlink for Local Testing (Go Packages Only)

For Go packages, the `package/.aux4` references the binary as `${packageDir}/<binary-name>`, which resolves to `package/<binary-name>`. During development and before running tests, create a symlink from `package/<binary-name>` pointing to the compiled binary for your current OS/architecture:

```bash
# Detect current OS and architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
case "$ARCH" in
  x86_64) ARCH="amd64" ;;
  aarch64|arm64) ARCH="arm64" ;;
  i386|i686) ARCH="386" ;;
esac

# Create symlink
ln -sf dist/${OS}/${ARCH}/<binary-name> package/<binary-name>
```

This symlink is already excluded by the `.gitignore`. Always build the binary (`aux4 build`) before creating the symlink, and recreate it if you change OS/architecture.

## Preparing Go Packages for Testing

For Go packages (with `dist/` binaries), you must build and create a symlink before running tests. The `package/.aux4` references the binary as `${packageDir}/<binary-name>`, so a symlink must exist at `package/<binary-name>`:

```bash
# Build all platforms
aux4 build

# Create symlink for current platform
ln -sf dist/$(uname -s | tr '[:upper:]' '[:lower:]')/$(uname -m | sed 's/x86_64/amd64/;s/aarch64\|arm64/arm64/;s/i[36]86/386/')/<binary-name> package/<binary-name>

# Run tests (from package/ directory)
cd package && aux4 test run
```

## Preparing JavaScript Packages for Testing

For JS packages, always build before testing to ensure the bundle is up to date:

```bash
# Build the bundle
npm run build

# Run tests (from package/ directory)
cd package && aux4 test run
```

**IMPORTANT:** Always run `aux4 test run` from the `package/` or `package/test/` directory. If you get "Command not found", you are in the wrong directory.

## Build, Install Locally, and Publish

**Important**: All pkger and releaser commands must be run from the `package/` directory, not from the project root.

### Using pkger directly

```bash
cd package

# Build the package
aux4 aux4 pkger build ./out .

# Test locally from file
aux4 aux4 pkger install --fromFile ./out/scope_name_1.0.0.zip

# Login and publish
aux4 aux4 login
aux4 aux4 pkger publish ./out/scope_name_1.0.0.zip

# Others install from hub
aux4 aux4 pkger install scope/name
```

### Using package-releaser (recommended)

Install the releaser: `aux4 aux4 pkger install aux4/package-releaser`

```bash
cd package

# Install locally for testing (builds, bumps version to X-local, installs, restores version)
aux4 aux4 releaser install

# Release (increments version, builds, publishes, tags git, creates GitHub release)
aux4 aux4 releaser release --level patch

# Skip build step if already built
aux4 aux4 releaser install --noBuild true
aux4 aux4 releaser release --level minor --noBuild true
```

The `release` command does everything in one step:
1. `git pull -r` to get latest changes
2. Prompts for confirmation and increments version (patch/minor/major)
3. Runs `aux4 build` (unless `--noBuild true`)
4. Runs `aux4 aux4 pkger build` to create the zip
5. Runs `aux4 aux4 pkger publish` to upload to hub.aux4.io
6. Creates git tag and GitHub release with the zip attached
7. Cleans up the zip file

## Testing Long-Running Processes (Servers, Daemons)

When tests require a background server or daemon, use `beforeAll` to start it and `afterAll` to stop it. Use `nohup` with output redirection to prevent the hook from blocking:

````markdown
# queue create

```beforeAll
nohup aux4 queue start --port 18420 >/dev/null 2>&1 &
sleep 1
```

```afterAll
aux4 queue stop --port 18420
```

## should create a queue

```execute
aux4 queue create --name myqueue --port 18420
```

```expect
queue "myqueue" created
```
````

**Key points:**
- Use `nohup ... >/dev/null 2>&1 &` to background the process without blocking the `beforeAll` hook
- Add `sleep 1` after starting to allow the server time to initialize
- Always clean up in `afterAll` (stop server, remove temp files)
- Use unique ports per test file to avoid conflicts when running tests in parallel
