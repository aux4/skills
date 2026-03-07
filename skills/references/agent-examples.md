# Agent Examples

## Complete Example: Code Review Agent

### package/.aux4

```json
{
  "scope": "tools",
  "name": "code-reviewer",
  "version": "0.1.0",
  "description": "AI agent that reviews code for quality and best practices",
  "dependencies": ["aux4/ai-agent"],
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "review",
          "execute": ["profile:review"],
          "help": { "text": "Code review commands" }
        }
      ]
    },
    {
      "name": "review",
      "commands": [
        {
          "name": "check",
          "execute": [
            "aux4 ai agent ask --configFile ${packageDir}/config.yaml --config reviewer --instructions ${packageDir}/instructions.md \"review the code in ${path}\""
          ],
          "help": {
            "text": "Review code in a file or directory",
            "variables": [
              {
                "name": "path",
                "text": "File or directory to review",
                "arg": true
              }
            ]
          }
        },
        {
          "name": "diff",
          "execute": [
            "stdin:aux4 ai agent ask --configFile ${packageDir}/config.yaml --context --config reviewer --instructions ${packageDir}/instructions/diff-review.md \"review this diff\""
          ],
          "help": {
            "text": "Review a git diff from stdin"
          }
        }
      ]
    }
  ]
}
```

### package/config.yaml

```yaml
config:
  reviewer:
    model:
      type: openai
      config:
        model: gpt-5-mini
```

### package/instructions.md

```markdown
# Code Review Agent

You are a code review agent. Your job is to review code for quality, best practices, and potential issues.

## Process

1. Use `listFiles` to understand the project structure
2. Use `readFile` to read the files to review
3. Analyze the code for issues
4. Provide a clear, actionable review

## Review Checklist

- Code correctness and logic errors
- Security vulnerabilities (injection, XSS, etc.)
- Performance issues
- Error handling
- Code style and readability
- DRY principle violations

## Output Format

For each issue found:
- **File**: path and line number
- **Severity**: critical / warning / suggestion
- **Issue**: what's wrong
- **Fix**: how to fix it

If no issues found, say "No issues found."
```
