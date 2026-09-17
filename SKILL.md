---
name: git-on
description: Publish the current local project to GitHub when the user asks to upload, publish, or sync it. Before any upload, confirm repository visibility, license, and sensitive-file handling; then connect GitHub safely, commit, and push. Do not use for non-GitHub hosting or deployment-only requests.
---

# Git On

Turn one explicit publishing request into a complete safe GitHub upload. Always pause for one preflight reply before making changes or uploading. After the user answers, complete the safe workflow without repeated confirmations unless a new blocker appears.

## Mandatory First Reply

On the first turn after an upload request, perform only read-only inspection as needed, then ask the user to confirm all of these items in one concise message:

1. Repository visibility: `private` or `public`.
2. License: keep the existing license, add `MIT`, choose another named license, or use no license.
3. Sensitive-file handling: exclude detected local secrets and environment files with `.gitignore`, or stop and let the user handle them. List only suspected file paths; never show secret values.
4. When GitHub is not connected: whether Codex may start the secure GitHub sign-in flow.

Do not initialize Git, edit `.gitignore`, create or change a license, stage, commit, create a remote repository, or push until the user answers. If the project already has a remote, license, or visibility, include that finding so the user can confirm whether to preserve it.

## Defaults

- Treat the current workspace as the project. If it is inside a Git repository, use the repository root.
- Reuse `origin` and its current visibility when it exists.
- When no remote repository exists, create one with the project folder name and default branch `main`, using the visibility confirmed in the preflight reply.
- Preserve an existing repository's visibility unless the user explicitly confirms a change.
- Preserve an existing license unless the user explicitly confirms a change. Add the exact standard license selected by the user only when no license exists or replacement was explicitly requested.
- Preserve existing README, license, branch history, remotes, and project structure. Do not generate extra files unless required to prevent sensitive files from being tracked.
- Use Git and GitHub CLI (`gh`). Do not require a browser or GitHub plugin when the CLI can finish the task.

## Workflow

1. Resolve the exact project root. Refuse drive roots, home directories, or other broad locations that are not clearly a project.
2. Read-only inspect `git status`, existing remotes, current branch, license files, `.gitignore`, and likely sensitive files. Preserve unrelated work inside the chosen project; it is part of the requested upload unless the user excluded it.
3. Ask the mandatory preflight questions above and stop the turn. Continue only after the user answers.
4. Before staging, inspect candidate file names and use filename-only searches for likely credentials. Never print secret values. At minimum detect:
   - `.env`, `.env.*`, `secrets.toml`, credential JSON files, private keys, certificates with private material, and local key stores.
   - Common token prefixes and assignments such as OpenAI keys, GitHub tokens, cloud access keys, `API_KEY=`, `TOKEN=`, `PASSWORD=`, and private-key headers.
   - Build outputs, virtual environments, caches, editor state, and dependency folders that should normally be ignored.
5. Apply the user's confirmed sensitive-file choice. Add only necessary ignore entries for confirmed local or sensitive artifacts. If a suspected secret is already tracked, stop and identify the file without displaying its contents. Do not rewrite Git history automatically.
6. Apply the confirmed license choice with the smallest necessary change.
7. If the project is itself a Codex Skill, run the available `quick_validate.py` against its folder before publishing. Stop on validation errors.
8. Initialize Git only when needed. Use or rename the initial branch to `main`; do not rename an established branch.
9. Stage the current project with `git add -A`. Review `git diff --cached --stat` and the staged file list before committing. Do not proceed if files resolve outside the project root.
10. If staged changes exist, create one concise Conventional Commit:
   - New publication: `feat: publish project`
   - Existing project update: summarize the actual staged change.
   If nothing changed, skip the commit and continue to remote verification.
11. If `origin` exists, fetch it and confirm the target branch can be pushed without rewriting remote history. If it does not exist, require an authenticated `gh` session and create the repository using the confirmed settings.
12. Push normally and set upstream when needed. Never use force push.
13. Verify the remote branch points to the local commit, then report the repository URL, visibility, license, branch, commit hash, and any skipped files.

## GitHub Account Connection

- First check `gh auth status`. Reuse an authenticated account after showing its username in the preflight question.
- If no account is connected and the user approved sign-in, use GitHub CLI's secure browser or device authorization flow, such as `gh auth login --web`.
- GitHub account passwords are not supported for Git operations. Never ask the user to paste a GitHub password, personal access token, or recovery code into chat, a command argument, a file, or tool output.
- If browser sign-in is unavailable, instruct the user to create a personal access token and enter it only through GitHub CLI's local secure prompt. Continue after `gh auth status` succeeds.
- Never store credentials in the project, commit them, echo them, or upload them.

## Stop Conditions

Stop and ask for the minimum required user action when:

- A secret or credential may be tracked.
- The user has not answered every mandatory preflight item.
- `gh` is missing or GitHub authentication is unavailable and no usable remote exists.
- The remote target is ambiguous or points to an unexpected repository.
- The remote branch has diverged, a merge/rebase is active, or pushing would require force.
- Repository visibility, license choice, or sensitive-file handling is ambiguous.
- The project root is unsafe or unclear.

Never run `git push --force`, `git reset --hard`, destructive checkout commands, history rewriting, repository deletion, or visibility changes.

## Completion Message

Keep the result short. Include:

```text
已上传：<repository URL>
分支：<branch>
提交：<short hash> <subject>
可见性：<private/public/unchanged>
```

If stopped, state the exact blocker and the single action needed to continue.
