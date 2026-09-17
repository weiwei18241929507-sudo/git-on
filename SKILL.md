---
name: git-on
description: Publish the current local project to GitHub when the user says "把这个项目上传到github上" or otherwise explicitly asks to upload, publish, or sync the current project to GitHub. Reuse an existing Git remote or create a private repository, check for secrets, commit project changes, and push safely. Do not use for non-GitHub hosting or deployment-only requests.
---

# Git On

Turn one explicit publishing request into a complete safe GitHub upload. The user's request to upload or publish is authorization to initialize Git, create a private repository when needed, commit the current project, and push it. Do not ask for redundant confirmation unless the requested action materially exceeds these defaults.

## Defaults

- Treat the current workspace as the project. If it is inside a Git repository, use the repository root.
- Reuse `origin` and its current visibility when it exists.
- When no remote repository exists, create one with the project folder name, default visibility `private`, and default branch `main`.
- Create a public repository only when the user explicitly says `公开` or `public`.
- Preserve existing README, license, branch history, remotes, and project structure. Do not generate extra files unless required to prevent sensitive files from being tracked.
- Use Git and GitHub CLI (`gh`). Do not require a browser or GitHub plugin when the CLI can finish the task.

## Workflow

1. Resolve and print the exact project root. Refuse drive roots, home directories, or other broad locations that are not clearly a project.
2. Inspect `git status`, existing remotes, current branch, and `.gitignore`. Preserve unrelated work inside the chosen project; it is part of the requested upload unless the user excluded it.
3. Before staging, inspect candidate file names and use filename-only searches for likely credentials. Never print secret values. At minimum detect:
   - `.env`, `.env.*`, `secrets.toml`, credential JSON files, private keys, certificates with private material, and local key stores.
   - Common token prefixes and assignments such as OpenAI keys, GitHub tokens, cloud access keys, `API_KEY=`, `TOKEN=`, `PASSWORD=`, and private-key headers.
   - Build outputs, virtual environments, caches, editor state, and dependency folders that should normally be ignored.
4. Add only necessary ignore entries for confirmed local or sensitive artifacts. If a suspected secret is already tracked, stop and identify the file without displaying its contents. Do not rewrite Git history automatically.
5. If the project is itself a Codex Skill, run the available `quick_validate.py` against its folder before publishing. Stop on validation errors.
6. Initialize Git only when needed. Use or rename the initial branch to `main`; do not rename an established branch.
7. Stage the current project with `git add -A`. Review `git diff --cached --stat` and the staged file list before committing. Do not proceed if files resolve outside the project root.
8. If staged changes exist, create one concise Conventional Commit:
   - New publication: `feat: publish project`
   - Existing project update: summarize the actual staged change.
   If nothing changed, skip the commit and continue to remote verification.
9. If `origin` exists, fetch it and confirm the target branch can be pushed without rewriting remote history. If it does not exist, require an authenticated `gh` session and create the repository using the defaults above.
10. Push normally and set upstream when needed. Never use force push.
11. Verify the remote branch points to the local commit, then report the repository URL, visibility when known, branch, commit hash, and any skipped files.

## Stop Conditions

Stop and ask for the minimum required user action when:

- A secret or credential may be tracked.
- `gh` is missing or GitHub authentication is unavailable and no usable remote exists.
- The remote target is ambiguous or points to an unexpected repository.
- The remote branch has diverged, a merge/rebase is active, or pushing would require force.
- Creating a public repository was not explicitly requested.
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
