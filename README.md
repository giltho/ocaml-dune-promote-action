# ocaml-dune-promote-action

A GitHub action for automatically running `dune promote` and committing only the promoted test files to a pull request.

## Features

- Runs `dune test` followed by `dune promote` to fix test outputs
- Parses dune promote output to identify exactly which files were promoted
- Commits only the promoted files (not all changed files)
- Pushes changes back to the PR branch
- Triggers only when a PR comment contains `!dune-promote`

## Usage

### Basic Setup

1. Add this action to your workflow file (e.g., `.github/workflows/dune-promote.yml`):

```yaml
name: Dune Promote on Comment

on:
  issue_comment:
    types: [created]

jobs:
  dune-promote:
    # Only run on PR comments with the trigger phrase
    if: github.event.issue.pull_request && contains(github.event.comment.body, '!dune-promote')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    
    steps:
      - name: Get PR branch
        id: pr-branch
        uses: xt0rted/pull-request-comment-branch@v2
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Checkout PR branch
        uses: actions/checkout@v4
        with:
          ref: ${{ steps.pr-branch.outputs.head_ref }}
          token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Set up OCaml
        uses: ocaml/setup-ocaml@v2
        with:
          ocaml-compiler: 4.14.x
      
      - name: Install dependencies
        run: opam install . --deps-only --with-test
      
      - name: Run Dune Promote
        uses: giltho/ocaml-dune-promote-action@v1  # Pin to a specific version tag
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Comment on PR
        if: success()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Dune promote completed successfully!'
            })
```

**Important Security Note**: Always pin the action to a specific version tag (e.g., `@v1`, `@v1.0.0`) or commit SHA in your workflow. Do not use `@main` or `./` in production workflows triggered by PR comments, as this could allow untrusted code execution if a malicious PR modifies the action code.

2. To trigger the action, comment on a PR with:
   ```
   !dune-promote
   ```

The action will run `dune test`, then `dune promote` to fix any test output mismatches, and commit only the promoted files to the PR.

## Inputs

| Input | Description | Required |
|-------|-------------|----------|
| `github-token` | GitHub token for committing and pushing changes | Yes |

## How it Works

1. The workflow is triggered when a comment is created on a pull request
2. It checks if the comment contains `!dune-promote`
3. If triggered, it:
   - Checks out the PR branch
   - Sets up the OCaml environment
   - Runs `dune test` (which may fail with test mismatches)
   - Runs `dune promote` and captures the output showing which files are promoted
   - Commits and pushes only the promoted files back to the PR

The example workflow also posts a success comment on the PR after the action completes.

Example output from `dune promote`:
```
Promoting _build/default/test/test.exe.output to test/test.expected
```

## Requirements

- An OCaml project using Dune as the build system
- Tests that can be promoted with `dune promote`
- GitHub Actions enabled in your repository

## Permissions

The workflow needs the following permissions:
- `contents: write` - to push commits
- `pull-requests: write` - to comment on PRs

Make sure your `GITHUB_TOKEN` has these permissions in your workflow.

## Security Considerations

This action is designed to run on pull request code, which inherently involves some security considerations:

1. **Pin to specific versions**: Always reference this action using a specific version tag (e.g., `@v1`) or commit SHA, never use `@main` or `./` in production workflows triggered by PR comments.

2. **Trusted repositories**: This action is most appropriate for repositories where PR authors are trusted (e.g., internal team repositories, or repos with strict PR review requirements).

3. **Review before merging**: The action only commits the promoted test files; it doesn't automatically merge them. Reviewers should always check the promoted changes before merging the PR.

4. **Limited scope**: The action only runs `dune test --auto-promote` and commits the results. It doesn't execute arbitrary code beyond what your test suite already does.

If you're concerned about security, consider:
- Requiring PR approval before the action can be triggered
- Using CODEOWNERS to control who can approve PRs
- Limiting who can trigger the action by checking the commenter's permissions in the workflow

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
