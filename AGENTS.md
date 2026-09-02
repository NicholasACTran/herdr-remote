# Agent Instructions

## Guidelines

- Read the project README and any existing docs before making changes
- Run the project's build/test commands before committing (check package.json, Makefile, pyproject.toml, Cargo.toml)
- Keep changes minimal and focused on the task
- Prefer early returns over nested conditionals
- Handle error states explicitly
- Use semantic HTML and ARIA attributes for accessibility in frontend code
- Follow existing code style and conventions in the repo
- Do not introduce new dependencies without justification

## Verification

- Run linting and type checks before committing
- Run tests relevant to changed code
- Verify the build passes

## Git

- Write clear, concise commit messages
- Stage only files related to the current task
- Do not push to main/master without explicit permission
- This repo is worked from a fork (`origin`) with the original author's repo
  as `upstream`. `gh pr create` with no explicit target defaults its base to
  the fork's *parent* (upstream), not `origin` - it assumes you're
  contributing back. To open a PR against the fork itself, pass
  `--repo <fork-owner>/<repo>` explicitly (and `--base`/`--head` as plain
  branch names, since head and base are then in the same repo). Verify with
  `gh pr view <n> --json isCrossRepository` before reporting a PR URL as
  done - `false` means it landed on the intended repo.

## Remote access tunnel backends

`HERDR_TUNNEL_MODE` (in `~/.config/herdr-remote/config.env`) selects how
the relay is reached remotely: `temp`/`named` (Cloudflare Tunnel, the
default, driven by `relay/install-service.sh`) or `aws` (a reverse SSH
tunnel to a self-hosted EC2 host, driven by `relay/tunnel-aws.sh`; see
`infra/aws-tunnel/README.md` for the host definition, cost, and deploy
steps). `relay/herdr-remote` is the canonical `start/stop/status` wrapper
and supports both. The relay itself always stays bound to loopback
(`relay/herdr_relay.py`) regardless of tunnel backend.

In `aws` mode, `relay/aws-fw-heal.sh` finds the tunnel's EC2 instance by
its CloudFormation tags (`project=herdr-remote`, `Name=herdr-remote-tunnel`)
rather than a hard-coded instance/security-group id - do the same in any
future AWS-tunnel tooling. It only ever adds an SSH ingress `/32`, never
revokes one; see its header comment before changing that.

## Maintaining this file

Keep this file for knowledge useful to almost every future agent session in this project.
Do not repeat what the codebase already shows; point to the authoritative file or command instead.
Prefer rewriting or pruning existing entries over appending new ones.
When updating this file, preserve this bar for all agents and keep entries concise.
