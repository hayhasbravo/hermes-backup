# cloud-dev-environment

Manage a cloud code environment (GitHub Codespaces / VPS) end to end: assess it,
decide whether to deploy, keep it alive across resets, and auto-backup so nothing
needs to be reinstalled by hand.

## When to use
- "Can I deploy a website here?"
- "Will my environment reset, and when?"
- "How do I back up this environment so a 30-day expiry doesn't mean reinstalling everything?"
- Any task that touches environment assessment, deployment, or backup.

---

## Procedure (do these, in order)

1. **Assess the environment.** Read resources (CPU/RAM/disk) and what tools exist
   (Docker, Node, Python, web servers). Decide if it is a dev workspace or a
   production server. This tells you the deployment path and what can be backed up.
2. **Decide the deployment path.**
   - Dev workspace with Docker → containerize with a Dockerfile.
   - No web server installed → containerize or install one.
   - Pure static front-end with no server → static host (Vercel/Netlify).
3. **Persist source.** Put every file worth keeping into git and push to a remote.
   Codespace workspace files survive stop/resume, but are **deleted on dispose/expiry**.
4. **Add backup + restore automation.** Create `backup.sh` + `restore.sh` and a
   monthly GitHub Actions snapshot workflow.
5. **Verify.** Run `bash backup.sh` once (confirm it writes), then confirm the
   scheduled action runs successfully. That is the "active" evidence.

## Standing preferences
- Concise, structured answers: short bullets, resources first, no narration of tool
  calls the user can see.
- When you need credentials, request ONE thing at a time and stop rather than guessing.
- When automating GitHub (push/schedule), use a **classic** PAT with `repo` scope.
  A fine-grained PAT can `gh auth` yet still fail `git push` / Action scheduling --
  it is read-only for that repository, and the failure is silent (auth passes, write
  is denied). Before scheduling, test both:
  `git ls-remote` (read) and `git push` (write), not just `gh auth status`.

## References
- `references/backup-and-deploy-recipe.md` — full backup.sh / restore.sh / GitHub
  Actions monthly-snapshot template.

## Pitfalls
- **Codespace "backup" is wrong.** Workspace files in the persisted share survive
  stop/resume only. Entire codespace goes away on expiry/disposal → back up git +
  rebuild script, not the container.
- **Never archive the source dir in its own location.** tar warns "file changed as we
  read it" and exits 1. Write the archive to `/tmp` or a sibling directory, never into
  the folder being tar'd.
- **GitHub Actions runs clean.** It reproduces the app from the repo, but cannot see
  the codespace's in-memory global packages or baked files. That is fine: the workflow's
  job is to prove the stack is rebuildable, and the artifact is a 30-day snapshot.
- **`commit.gpgsign=true` blocks commits.** Disable it (`git config commit.gpgsign false`)
  or record a GPG key before pushing.
- **No GitHub auth, no workflow.** `gh auth` (or a repo-scoped PAT as `GH_TOKEN`) is
  required to push and schedule the action. Ask for it; do not guess a token.
- **Authenticated ≠ writable.** If `git ls-remote` (read) succeeds but `git push` or
  Action publishing fails with 403, the token is fine-grained-and-read-only for that
  repository. Do not schedule Actions until you obtain a classic PAT with `repo` scope
  and re-test read + write.

## Deployment decision table

| Environment | Tools present | Best move |
|---|---|---|
| Codespace (ephemeral) | docker, node, python | containerize + monthly auto-backup |
| VPS / production server | nginx, systemd, Docker | run as a service |
| No server, pure static | only the repo | static host (Vercel / Netlify) |
