---
name: code-sleuth
version: 1.2.0
author: Kilo Engineering Team
description: Intelligently analyzes codebases to detect patterns, anti-patterns, and improvement opportunities using static analysis, pattern matching, and code metrics
tags:
  - code analysis
  - refactoring
  - patterns
  - anti-patterns
  - code quality
dependencies:
  - cli: ["rg", "ag", "find", "sed", "awk", "jc", "jq", "yamllint", "hadolint", "shellcheck", "eslint", "pylint", "golangci-lint"]
  - python: ["radon", "bandit", "prospector", "vulture"]
  - node: ["depcheck", "npx"]
  - go: ["gocyclo", "go-vet"]
  - java: ["pmd", "checkstyle"]
  - ruby: ["rubocop"]
  - rust: ["cargo-audit", "clippy"]
  - php: ["phpstan", "psalm"]
required_env_vars: []
working_directory: repository root
interactive: false
timeout: 300000
---

# Code Sleuth

## Purpose

Code Sleuth performs deep, intelligent analysis of codebases to identify design patterns, code smells, security vulnerabilities, performance bottlenecks, and refactoring opportunities. It provides actionable insights with specific file locations and remediation suggestions.

### Real Use Cases

1. **Security Audit**: Scan a Node.js application for common vulnerabilities (XSS, SQL injection, command injection) before deployment
2. **Technical Debt Assessment**: Quantify cyclomatic complexity, duplicated code, and long methods across a Python codebase to prioritize refactoring
3. **Code Freeze Detection**: Identify files with excessive changes and high churn rates to spot instability
4. **Dependency Analysis**: Find unused dependencies and security vulnerabilities in package.json, requirements.txt, or go.mod
5. **Architecture Compliance**: Verify that code adheres to architectural constraints (e.g., no circular dependencies, proper layering)
6. **Performance Investigation**: Locate potential N+1 queries, inefficient algorithms, and resource leaks
7. **Onboarding Acceleration**: Generate codebase health reports for new developers

## Scope

### Supported Languages & Tools

**Multi-language static analysis**:
- JavaScript/TypeScript: ESLint with security plugins (eslint-plugin-security, eslint-plugin-no-unsanitized)
- Python: Bandit (security), Radon (complexity), Pylint, Vulture (dead code)
- Go: govet, gocyclo, staticcheck
- Java: PMD, Checkstyle, SpotBugs
- Ruby: RuboCop with security cops
- Rust: Clippy, cargo-audit
- PHP: PHPStan, Psalm
- Shell: ShellCheck
- Docker: Hadolint
- YAML/JSON: yamllint, jsonlint

**Pattern Detection**:
- God classes (single responsibility violations)
- Feature envy (methods using data from other classes excessively)
- Data clumps (groups of parameters passed together)
- Switch statements (indicator of missing polymorphism)
- Long parameter lists (>3 parameters)
- Deep nesting (>3 levels)
- Duplicate code (copy-paste detection)
- Magic numbers and strings
- TODOs/FIXMEs in production code

**Architecture Analysis**:
- Circular dependency detection (using dependency graphs)
- Layer violation checking (UI→Business→Data access)
- Dead code identification (unused functions/classes)
- API contract stability (breaking changes detection)

### Specific Commands

Code Sleuth executes these commands internally:

```bash
# Complexity analysis
radon cc --total-average --show-complexity src/  # Python
gocyclo -over 15 ./...                           # Go

# Security scanning
bandit -r src/ -f json -o security-report.json
eslint src/ --plugin security --format json
shellcheck -x scripts/*.sh

# Duplicate code detection
jscpd src/ --format json --min-tokens 50

# Dependency analysis
depcheck                   # Node.js unused deps
go mod graph | sort | uniq -d  # Go circular deps

# Code churn analysis
git log --since="1 year ago" --pretty=format: --name-only | sort | uniq -c | sort -nr | head -20

# Architecture validation
jdepend --package=com.myapp src/main/java  # Java package cycles
import-linter src/ --config .importlinter.yml

# Pattern searching
rg -tpy "class.*:.*\n(.*\n){30,}" --max-count 100  # God class heuristic
rg "TODO|FIXME|HACK|XXX" --line-number --context 2 --ignore-case

# File size analysis
find src/ -name "*.py" -exec wc -l {} + | sort -rn | head -20
```

## Work Process

### Phase 1: Environment Discovery (1-2 minutes)

1. **Detect project type** by examining files:
   ```bash
   test -f package.json && echo "node"
   test -f requirements.txt && echo "python"
   test -f go.mod && echo "go"
   test -f pom.xml && echo "java"
   ```
   
2. **Load language-specific linters** from `.codesleuth.yml` or auto-detect
3. **Parse git history** for ownership and churn metrics:
   ```bash
   git log --pretty=format:"%an" --since="6 months ago" | sort | uniq -c | sort -rn > owners.txt
   ```

### Phase 2: Static Analysis (5-15 minutes)

Run parallel analysis jobs:

```bash
# Job 1: Security vulnerabilities
bandit -r src/ -f json -o /tmp/bandit.json &
eslint src/ --format json > /tmp/eslint.json &

# Job 2: Code complexity
radon cc src/ -j > /tmp/radon.json &
gocyclo -over 10 ./... > /tmp/gocyclo.txt &

# Job 3: Pattern detection
rg --json "TODO|FIXME|HACK" src/ > /tmp/todos.json &
depcheck --json > /tmp/depcheck.json &

wait
```

### Phase 3: Pattern Correlation (2-3 minutes)

Apply correlation rules:

```python
# Example: Correlate high complexity with low test coverage
for file in complex_files:
    if file not in test_files:
        flag(file, "high complexity with no tests")

# Example: Flag functions modified by >3 developers (bus factor)
if len(owners[file]) > 3 and churn[file] > 100:
    flag(file, "high bus factor risk")
```

### Phase 4: Report Generation (1 minute)

Generate JSON report with findings grouped by:
- Severity: CRITICAL, HIGH, MEDIUM, LOW, INFO
- Category: security, complexity, architecture, performance, maintainability
- File location with line numbers
- Code snippet (3 lines context)
- Recommended fix with example

### Phase 5: Interactive Summary (optional)

If `--interactive` flag set:
```bash
less +G reports/findings-$(date +%Y%m%d).md
```

## Golden Rules

1. **Never modify source code** - read-only analysis only. Use `--fix` flag explicitly for automated fixes.
2. **Always show exact line numbers** - include file path, line, and column in all findings.
3. **Suppress false positives** - respect `.codesleuthignore` patterns and inline `// codesleuth-ignore` comments.
4. **Prioritize by impact** - security issues > crashing bugs > performance > code smell > style.
5. **Provide actionable remediation** - every finding must include specific code change or documentation reference.
6. **Respect CI/CD timeouts** - if `CI=true` env var, skip non-essential analysis (e.g., duplicate detection).
7. **Cache results** - store analysis in `.codesleuth/cache/` keyed by git commit SHA; re-run only on changed files when possible.
8. **Respect git blame boundaries** - don't analyze generated code or vendor directories (auto-detected).
9. **Zero tolerance for crashes** - linter crashes on one file shouldn't stop entire analysis.
10. **Report only on tracked code** - exclude untracked files unless `--include-untracked` flag.

## Examples

### Example 1: Security audit before release

**User command**:
```bash
codesleuth analyze --security --format sarif --output security.sarif
```

**What Code Sleuth does**:
```bash
# Bandit scan
bandit -r src/ -f json -o /tmp/bandit.json

# ESLint security rules
eslint src/ --plugin security --format json > /tmp/eslint.json

# Convert to SARIF
codesleuth convert /tmp/bandit.json --to sarif --output security.sarif
```

**Output structure**:
```json
{
  "tool": "CodeSleuth",
  "timestamp": "2026-01-15T14:32:10Z",
  "findings": [
    {
      "severity": "CRITICAL",
      "category": "security",
      "file": "src/auth.py",
      "line": 42,
      "message": "SQL injection vulnerability: raw SQL query with string interpolation",
      "snippet": "query = f\"SELECT * FROM users WHERE id = '{user_id}'\"",
      "recommendation": "Use parameterized queries: cursor.execute(\"SELECT * FROM users WHERE id = %s\", (user_id,))",
      "cwe": "CWE-89"
    }
  ]
}
```

### Example 2: Technical debt assessment

**User command**:
```bash
codesleuth debt --threshold 15 --hotspots --format markdown
```

**What Code Sleuth does**:
```bash
# Find functions with complexity > 15
radon cc src/ -j | jq -r 'to_entries[] | select(.value.complexity > 15) | "\(.key):\(.value.lineno): \(.value.complexity)"' > /tmp/complex.txt

# Find files with >500 lines
find src/ -name "*.py" -exec wc -l {} + | awk '$1 > 500 {print $2 ":" $1}' > /tmp/long_files.txt

# Generate hotspots table
echo "| File | Line | Complexity | Lines | Developers | Last Modified |"
echo "|------|------|------------|-------|------------|---------------|"
while read file line complexity; do
    lines=$(wc -l < "$file")
    owners=$(git log --format="%an" -- "$file" | sort | uniq | wc -l)
    last_modified=$(git log -1 --format=%cd --date=relative -- "$file")
    echo "| $file | $line | $complexity | $lines | $owners | $last_modified |"
done < /tmp/complex.txt | sort -t'|' -k3 -rn > debt-report.md
```

**Sample output**:
```
| File | Line | Complexity | Lines | Developers | Last Modified |
| src/server/order_processor.py | 127 | 42 | 847 | 8 | 2 days ago |
| src/payments/gateway_manager.py | 89 | 38 | 623 | 5 | 1 week ago |
```

### Example 3: Architecture validation

**User command**:
```bash
codesleuth arch --layers --detect-cycles
```

**What Code Sleuth does**:
```bash
# Build dependency graph for Java
jdepend --package=com.myapp src/main/java > /tmp/deps.txt

# Check for circular dependencies
awk '/cycle/ {print}' /tmp/deps.txt

# Verify layer rules (import-linter)
import-linter src/ --config .importlinter.yml --json > /tmp/layer_violations.json

# Generate PlantUML diagram
codesleuth visualize --format plantuml --output arch.puml
```

**Architecture finding**:
```json
{
  "violation": "cross_layer_import",
  "from": "com.myapp.ui.UserController",
  "to": "com.myapp.persistence.UserRepository",
  "layers": ["ui", "persistence"],
  "message": "UI layer directly importing persistence layer, bypassing service layer"
}
```

### Example 4: Unused dependency detection

**User command**:
```bash
codesleuth deps --unused --update-package-json
```

**What Code Sleuth does**:
```bash
# Run depcheck
depcheck --json > /tmp/depcheck.json

# Parse unused deps
jq -r '.dependencies | keys[]' /tmp/depcheck.json > /tmp/unused.txt

# Interactive confirmation
echo "Unused dependencies detected:"
cat /tmp/unused.txt
read -p "Remove these from package.json? (y/N): " confirm
if [[ $confirm == "y" ]]; then
    codesleuth deps --remove --package-json package.json --deps "$(cat /tmp/unused.txt)"
fi
```

## Rollback Commands

### Rollback analysis results
```bash
# Restore previous analysis results from git
git checkout HEAD~1 -- reports/

# Clear cache
rm -rf .codesleuth/cache/

# Reset configuration to defaults
cp .codesleuth.example.yml .codesleuth.yml
```

### Rollback automated fixes
```bash
# Review changes made by --fix flag
git status
git diff src/

# Revert all fixes from last run
git reset --hard HEAD

# For selective rollback (interactive)
git add -p  # stage hunks to keep
git checkout -- .  # discard unstaged changes
```

### Rollback dependency changes
```bash
# Undo package.json modifications
git checkout package.json package-lock.json

# Restore exact dependency versions from lockfile
npm ci  # or yarn install --frozen-lockfile

# Revert to previous dependency tree
git checkout vendor/  # if vendoring is used
```

### Cleanup residual artifacts
```bash
# Remove all Code Sleuth generated files
find . -name "*.codesleuth" -delete
find . -name ".codesleuth" -type d -exec rm -rf {} +
find . -name "*-codesleuth.*" -delete

# Remove temporary analysis directories
rm -rf /tmp/codesleuth-*
rm -rf reports/cache/
```

### Emergency stop during long-running analysis
```bash
# Find and kill Code Sleuth processes
pkill -f codesleuth
pkill -f radon
pkill -f bandit

# Cleanup partial results
rm -rf /tmp/codesleuth-*
```

### Restore original git state after analysis
```bash
# If analysis modified any files (shouldn't happen but for safety)
git reset --hard
git clean -fdX  # remove untracked files (use with caution)
```

### Resume interrupted analysis
```bash
# Code Sleuth auto-saves progress to .codesleuth/progress.json
codesleuth resume --from-cache  # picks up where left off
# Or manually:
cat .codesleuth/progress.json | jq -r '.completed[]' | xargs -I{} echo "Already analyzed: {}"
```

## Troubleshooting

### Issue: "Command not found: eslint"
**Solution**: Install missing linter. For Node.js projects: `npm install --save-dev eslint eslint-plugin-security`

### Issue: Analysis hangs on large file
**Solution**: Exclude pattern in `.codesleuthignore`:
```
# Ignore generated files
*.min.js
*.generated.py
vendor/
```

### Issue: False positive on CWE-78 (command injection)
**Solution**: Add inline suppression: `# codesleuth-ignore: CWE-78` on the line, or configure in `.codesleuth.yml`:
```yaml
suppressions:
  - file: "src/utils/safe_shell.py"
    rule: "CWE-78"
    reason: "All inputs are validated via validate_input()"
```

### Issue: Memory exhaustion on huge codebase
**Solution**: Limit analysis scope:
```bash
codesleuth analyze --max-files 1000 --exclude-dir node_modules,dist,build
# Or analyze incrementally:
codesleuth analyze --changed-only  # only files changed in current branch
```

### Issue: Incorrect complexity metrics for Python decorators
**Solution**: Radon doesn't handle decorators properly. Use `--ignore-decorators` flag or upgrade to radon>=5.0:
```bash
pip install --upgrade radon
```

### Issue: Linter version conflicts
**Solution**: Use Code Sleuth's isolated environment:
```bash
# Let codesleuth manage dependencies
codesleuth setup --isolated
# This creates .codesleuth/venv/ or .codesleuth/node_modules/
```

### Issue: Permission denied on git commands
**Solution**: Code Sleuth requires git repository access. Ensure:
```bash
git rev-parse --show-toplevel  # returns valid path
ls -la .git/  # directory accessible
# If using sudo, avoid it; fix permissions instead:
sudo chown -R $(whoami) .git/
```

### Issue: False cycle detection in Java due to test dependencies
**Solution**: Configure import-linter to exclude test sources:
```yaml
# .importlinter.yml
exclude:
  - "**/test/**"
  - "**/*Test.java"
```

### Issue: ESLint crashes on TypeScript syntax errors
**Solution**: Enable TypeScript parser:
```bash
npm install --save-dev @typescript-eslint/parser @typescript-eslint/eslint-plugin
# Then in .eslintrc:
{
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint"]
}
```

### Issue: Slow analysis on CI
**Solution**: Reduce analysis scope in CI:
```bash
# .gitlab-ci.yml or .github/workflows/ci.yml
codesleuth analyze --ci --max-workers 2 --skip-duplicate-detection
# This enables caching and parallelism optimizations
```

### Issue: Missing findings in vendored dependencies
**Solution**: Exclude vendor directories explicitly:
```bash
codesleuth analyze --exclude "vendor/,third_party/,external/"
# Or in .codesleuth.yml:
exclude:
  - vendor/**
  - third_party/**
```

### Issue: Cannot detect patterns in minified/obfuscated code
**Solution**: Code Sleuth auto-skips files with:
- Compression ratio > 60%
- Average line length > 200 chars
- Hex-encoded strings > 80% of content
Override with `--no-heuristic-filter` (not recommended).

### Issue: Different results between local and CI
**Solution**: Ensure consistent environment:
```bash
# Pin linter versions in .codesleuth.yml
tools:
  bandit: "1.7.4"
  radon: "5.1.0"
  eslint: "8.0.0"
```

### Issue: Output format unsupported by CI system
**Solution**: Convert to supported format:
```bash
codesleuth analyze --format json > findings.json
codesleuth convert findings.json --to sarif --output findings.sarif  # for GitHub CodeQL
codesleuth convert findings.json --to checkstyle --output findings.xml  # for Jenkins
```

### Issue: "No module named 'radon'" after Python 3.12 upgrade
**Solution**: Reinstall with proper Python version:
```bash
python3.12 -m pip install --user radon bandit
# Or use codesleuth's isolated env:
codesleuth setup --python /usr/bin/python3.12
```

### Performance tuning for >1M LOC codebases
```bash
# Use incremental analysis based on git diff
codesleuth analyze --base-ref main --head-ref feature-branch

# Disable expensive checks
codesleuth analyze --no-duplicate-detection --no-code-churn --fast

# Increase parallelism (if CPU cores available)
codesleuth analyze --max-workers $(nproc)
```

## Configuration

### `.codesleuth.yml` (example)

```yaml
exclude:
  - "**/node_modules/**"
  - "**/dist/**"
  - "**/build/**"
  - "**/*.min.js"
  - "**/vendor/**"
  - "**/migrations/**"

rules:
  complexity:
    python:
      threshold: 10
      tool: radon
    javascript:
      threshold: 15
      tool: eslint complexity
  security:
    enabled: true
    tools: [bandit, eslint-security, shellcheck]
  dead_code:
    enabled: true
    python: vulture
    javascript: depcheck
  architecture:
    java:
      enforce_layers: true
      max_cycle_length: 3

suppressions:
  - file: "**/test/**"
    reason: "test code exempt"
  - file: "src/legacy/"
    rule: "GOD_CLASS"
    reason: "Refactor planned in Q2 2026"

reporting:
  format: [json, markdown, sarif]
  output_dir: "reports"
  include_snippets: true
  snippet_lines: 3
```

### `.codesleuthignore`

```
# Patterns (gitignore syntax)
*.generated.js
*.min.*
coverage/
docs/
**/__pycache__/
**/*.pyc
**/node_modules/
**/venv/
**/target/
**/.tox/
```

## Integration

### GitHub Actions

```yaml
name: Code Quality
on: [push, pull_request]

jobs:
  codesleuth:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Code Sleuth
        run: |
          curl -fsSL https://get.kilo.ai/codesleuth | sh
          codesleuth setup
      - name: Analyze
        run: |
          codesleuth analyze --format sarif --output results.sarif
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: results.sarif
```

### Git Hook (pre-commit)

```bash
# .git/hooks/pre-commit
#!/bin/bash
codesleuth analyze --changed-only --severity-threshold HIGH
if [[ $? -ne 0 ]]; then
    echo "High severity issues detected. Fix or use --no-verify to skip."
    exit 1
fi
```

### CI/CD Pipeline (Jenkins)

```groovy
stage('Code Analysis') {
    steps {
        sh 'codesleuth analyze --format checkstyle --output checkstyle.xml'
        recordIssues(
            tools: [checkStyle()],
            qualityGates: [[threshold: 10, type: 'TOTAL', unstable: true]]
        )
    }
}
```

## Metrics Tracked

- **Cyclomatic Complexity**: Average and distribution per function
- **Maintainability Index**: Per file and aggregate (0-100 scale)
- **Technical Debt**: Estimated hours to fix all issues
- **Code Churn**: Frequency of changes per file (commits/week)
- **Bus Factor**: Number of developers needed to maintain codebase
- **Security Score**: Based on OWASP Top 10 coverage
- **Test Coverage**: Lines covered by tests (if coverage data available)
- **Duplication Rate**: Percentage of duplicated code blocks
- **Dependency Health**: Outdated, vulnerable, unused dependencies

## Limitations

- Static analysis only - cannot detect runtime issues without execution traces
- False positives in dynamically-typed languages (Python, Ruby)
- Requires accessible git history (cannot analyze snapshot without history)
- Architecture rules require manual configuration per project
- Security scanning limited to known patterns (no AI-based detection)
- Performance analysis doesn't include profiling data

## Future Enhancements

- AI-powered code smell classification
- Automated refactoring suggestions with code diffs
- Cross-repository analysis for microservices
- Integration with issue trackers to auto-create tickets
- Performance profiling integration (py-spy, perf, async-profiler)
- Machine learning-based complexity prediction

---
```