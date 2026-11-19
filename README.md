# github-action-gitops-update

Reusable GitHub Action to update a YAML file and commit the result.

It sets one or more YAML keys to a new value, commits the change as `ci(gitops): <commit_message>`, and pushes it to `target_branch`. If another run pushed to the branch first, it fetches the latest branch, re-applies the edit and tries again.

## Usage

```yaml
permissions:
  contents: write

jobs:
  gitops-update:
    runs-on: ubuntu-latest
    outputs:
      sha: ${{ steps.gitops.outputs.sha }}
    steps:
      - name: GitOps Update and Commit
        id: gitops
        uses: masterpointio/github-action-gitops-update@<commit-sha> # vX.Y.Z
        with:
          new_value: ${{ inputs.image_tag }}
          yaml_key_paths: .components.terraform.my-app.vars.image_tag
          yaml_file_path: stacks/org/dev/us-east-1.yaml
          commit_message: "update my-app image_tag to ${{ inputs.image_tag }}"
```

## Inputs

| Name              | Required | Default                                         | Description                                                      |
| ----------------- | -------- | ----------------------------------------------- | ---------------------------------------------------------------- |
| `new_value`       | yes      |                                                 | New value for the given YAML keys                                |
| `yaml_key_paths`  | yes      |                                                 | Comma-separated list of yq paths to update, e.g. `.a.tag,.b.tag` |
| `yaml_file_path`  | yes      |                                                 | Path to the YAML file to update                                  |
| `commit_message`  | yes      |                                                 | Commit message, prefixed with `ci(gitops):`                      |
| `commit_username` | no       | `github-actions`                                | Commit author name                                               |
| `commit_email`    | no       | `github-actions@github.com`                     | Commit author email                                              |
| `target_branch`   | no       | `${{ github.event.repository.default_branch }}` | Branch to push to                                                |
| `github_token`    | no       | `${{ github.token }}`                           | Token used to push (needed when the target branch is protected)  |
| `max_attempts`    | no       | `5`                                             | Number of times to retry the push when the target branch moved   |

## Outputs

| Name       | Description                                                           |
| ---------- | --------------------------------------------------------------------- |
| `sha`      | SHA of the pushed commit                                              |
| `base_sha` | SHA of the pushed commit's parent (`target_branch` before the update) |

## Concurrency

Runs that target the same branch at the same time are safe: each push that is rejected is retried on top of the latest branch, with a short random delay. Don't put callers in a shared `concurrency` group to serialize them, because GitHub cancels older pending runs in a group and those updates are lost.

## Notes

- yq removes blank lines when it writes a file, so the action applies yq's output through `diff -B | patch` to keep them. A blank line directly next to an edited key can still be removed.
- If the value is unchanged, the action still pushes an empty commit.
