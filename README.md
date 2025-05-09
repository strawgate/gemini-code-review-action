# gemini-code-review-action
A container GitHub Action to review code using Gemini AI. This action provides two main functionalities:

1. Automatic Pull Request Review
2. Comment-Driven Code Review

## Pre-requisites
We have to set a GitHub Actions secret `GEMINI_API_KEY` to use the Gemini API so that we securely pass it to the Action.

## Inputs

### Common Inputs
- `gemini_api_key`: The Gemini API key to access the Gemini API [(GET MY API KEY)](https://makersuite.google.com/app/apikey).
- `github_token`: The GitHub token to access the GitHub API (You do not need to generate this Token!).
- `github_repository`: The GitHub repository to post a review comment.
- `github_pull_request_number`: The GitHub pull request number to post a review comment.
- `git_commit_hash`: The git commit hash to post a review comment.
- `model`: The model to generate a review comment. We can use a model which is available.
- `log_level`: The log level to print logs.

### Pull Request Review Specific Inputs
- `pull_request_diff`: The diff of the pull request to generate a review comment.
- `pull_request_chunk_size`: The chunk size of the diff of the pull request to generate a review comment.

### Comment-Driven Review Specific Inputs
- `github_comment`: The GitHub comment content to parse for commands.
- `include_extensions`: Comma-separated list of file extensions to include in review.
- `always_include_files`: Comma-separated list of files to always include in review.

## Example Usage

### 1. Automatic Pull Request Review
This workflow automatically reviews pull requests when they are opened, synchronized, or reopened.

```yaml
name: "Automatic PR Review by Gemini AI"

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v3
      
      - name: "Get diff of the pull request"
        id: get_diff
        shell: bash
        env:
          PULL_REQUEST_HEAD_REF: "${{ github.event.pull_request.head.ref }}"
          PULL_REQUEST_BASE_REF: "${{ github.event.pull_request.base.ref }}"
        run: |-
          git fetch origin "${{ env.PULL_REQUEST_HEAD_REF }}"
          git fetch origin "${{ env.PULL_REQUEST_BASE_REF }}"
          git checkout "${{ env.PULL_REQUEST_HEAD_REF }}"
          git diff "origin/${{ env.PULL_REQUEST_BASE_REF }}" > "diff.txt"
          {
            echo "pull_request_diff<<EOF";
            cat "diff.txt";
            echo 'EOF';
          } >> $GITHUB_OUTPUT
          
      - uses: rubensflinco/gemini-code-review-action@1.0.5
        name: "Code Review by Gemini AI"
        with:
          gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          github_repository: ${{ github.repository }}
          github_pull_request_number: ${{ github.event.pull_request.number }}
          git_commit_hash: ${{ github.event.pull_request.head.sha }}
          model: "gemini-1.5-pro-latest"
          pull_request_diff: ${{ steps.get_diff.outputs.pull_request_diff }}
          pull_request_chunk_size: "3500"
          extra_prompt: |
            Sempre responda em português brasileiro!
          log_level: "DEBUG"
```

### 2. Comment-Driven Code Review
This workflow allows users to trigger code reviews using comments on pull requests. Available commands:

- `gemini review all` - Reviews entire repository
- `gemini review diff` - Reviews only changed files
- `gemini suggest next steps` - Provides suggestions for next steps

```yaml
name: "Comment-Driven Code Review by Gemini AI"

on:
  issue_comment:
    types: [created]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v3
      
      - name: "Get PR number from comment"
        id: get_pr_number
        run: |
          if [[ "${{ github.event.issue.pull_request }}" ]]; then
            PR_NUMBER=$(echo "${{ github.event.issue.pull_request }}" | awk -F'/' '{print $NF}')
            echo "pr_number=$PR_NUMBER" >> $GITHUB_OUTPUT
          else
            echo "Not a pull request comment, skipping"
            exit 0
          fi
          
      - uses: rubensflinco/gemini-code-review-action@1.0.5
        name: "Code Review by Gemini AI"
        if: steps.get_pr_number.outputs.pr_number != ''
        with:
          gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          github_repository: ${{ github.repository }}
          github_pull_request_number: ${{ steps.get_pr_number.outputs.pr_number }}
          git_commit_hash: ${{ github.event.issue.pull_request.head.sha }}
          model: "gemini-1.5-pro-latest"
          github_comment: ${{ github.event.comment.body }}
          include_extensions: ".py,.js,.ts,.java,.go"
          always_include_files: "README.md,CONTRIBUTING.md"
          extra_prompt: |
            Sempre responda em português brasileiro!
          log_level: "DEBUG"
```

## Example Output
As a result of an execution of the Action, it posts a review comment to the pull request like the following image.
![An example comment of the code review](./docs/images/example.png)
