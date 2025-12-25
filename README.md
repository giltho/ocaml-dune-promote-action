# ocaml-dune-promote-action

A GitHub action for automatically running `dune test --auto-promote` and committing the resulting files to a pull request.

## Features

- Runs `dune test --auto-promote` to automatically fix test outputs
- Detects and commits any modified files
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
        uses: giltho/ocaml-dune-promote-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

2. To trigger the action, comment on a PR with:
   ```
   !dune-promote
   ```

The action will automatically run `dune test --auto-promote`, commit any changes, and push them to the PR.

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
   - Runs `dune test --auto-promote`
   - Detects any file changes
   - Commits and pushes the changes back to the PR
   - Posts a success comment

## Requirements

- An OCaml project using Dune as the build system
- Tests that can be promoted with `dune test --auto-promote`
- GitHub Actions enabled in your repository

## Permissions

The workflow needs the following permissions:
- `contents: write` - to push commits
- `pull-requests: write` - to comment on PRs

Make sure your `GITHUB_TOKEN` has these permissions in your workflow.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
