# GitHub Monorepo Subfolder Synchronizer

This folder contains a Python-only synchronizer. There is no BAT file and no PowerShell file.

## Files

- `sync_github.py` — the program you run.
- `config.json` — all user-editable configuration.
- `README.md` — all explanations and operating notes.

The first run creates `.sync_github_state/`, a hidden full Git cache of the monorepo. It is required for reliable history comparison and conflict detection.

## Run

Double-click `sync_github.py` when Windows associates `.py` files with Python, or run:

```bat
py sync_github.py
```

or:

```bat
python sync_github.py
```

The default `"pause_before_exit": true` keeps the console visible until Enter is pressed. From a terminal, `python sync_github.py --no-pause` overrides it for one run.

## Requirements

- Python 3.
- Git for Windows.
- Git Credential Manager, normally installed with Git for Windows.
- Permission to access the configured GitHub repository.

The program uses only Python's standard library. No `pip install` is needed.

## Configuration

Edit `config.json` before the first run:

```json
{
  "repository_url": "https://github.com/example/company-monorepo.git",
  "branch": "main",
  "remote_subfolder": "applications/hgpt",
  "github_username": "your-github-name",
  "commit_message": "Automatic folder synchronization",
  "login_mode": "auto",
  "first_sync_source": "auto",
  "create_empty_remote_folder": true,
  "empty_folder_marker": ".gitkeep",
  "allow_empty_local_delete": false,
  "network_timeout_seconds": 120,
  "authentication_timeout_seconds": 600,
  "pause_before_exit": true
}
```

JSON does not permit comments, so all field explanations are here.

## Mapping

The folder containing `sync_github.py` maps directly to `remote_subfolder` inside the monorepo.

For example, local `C:\Projects\HGPT\app.py` maps to `applications/hgpt/app.py` when:

```json
"remote_subfolder": "applications/hgpt"
```

The rest of the monorepo remains in the hidden cache and is not copied into the visible local folder.

## Missing remote folder

When the configured remote subfolder does not exist, the first synchronization creates it from the visible local folder and pushes a normal monorepo commit.

Git cannot store an entirely empty directory. With `create_empty_remote_folder` set to `true`, the program creates the configured marker file, normally `.gitkeep`.

## Private repository login

Recommended:

```json
"login_mode": "auto"
```

Modes:

- `auto` — use saved credentials and initiate GitHub/browser login when access fails.
- `always` — initiate login before every run.
- `never` — never initiate login automatically.

The credential is stored by Windows Credential Manager through Git Credential Manager. It is not stored in `config.json`. Do not place a password or personal access token in the JSON file.

The program checks whether the configured GitHub account has a saved credential. If none exists, it opens browser login and automatically falls back to GitHub's device-code flow when browser login is unavailable. If GitHub rejects an upload, the program retries authentication once and reports commands for replacing an invalid saved credential.

Public repositories are checked anonymously first. GitHub authentication is deferred until an upload requires write access, avoiding unnecessary Credential Manager prompts and verification waits for read-only operations.

## Git configuration

The repository URL, branch, mapped folder, and GitHub username come from `config.json`.
The commit name and email come from the global Git configuration. Configure them once with:

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

The program does not require `git remote add` or an `origin` remote.

## Synchronization behavior

The program compares Git history and file content, not Windows modification dates. It automatically:

- pushes local-only changes;
- downloads GitHub-only changes;
- combines compatible work from two computers;
- preserves unrelated monorepo folders;
- records additions, modifications, renames, and deletions;
- stops when both sides changed the same content incompatibly.

On a conflict, it aborts the attempted rebase, lists the conflicting monorepo files, preserves local commits, and leaves GitHub unchanged. It never force-pushes.

## Two computers

Use the same `repository_url`, `branch`, and `remote_subfolder` on both computers. Run the program after finishing on one computer and before starting on the other. Each computer has its own `.sync_github_state` cache and its own secure Windows credential.

## First synchronization

`first_sync_source` is used only before this local folder has shared synchronization history:

- `auto` — creates a missing remote folder, downloads into an empty local folder, accepts identical content, but stops if both sides already contain different content.
- `local` — treats the visible local folder as the intended first version.
- `github` — treats GitHub as the intended first version and creates a timestamped sibling backup before replacement.

After the first successful connection, this field is no longer used.

## Complete deletion safeguard

With `"allow_empty_local_delete": false`, an entirely empty visible local folder cannot delete every tracked remote file accidentally. Set it to `true` only for an intentional complete deletion.

## `.gitignore`

A `.gitignore` inside the visible local folder is synchronized into the mapped monorepo folder. Standard Git rules apply: new ignored files are not committed, while already tracked files remain tracked until explicitly removed from tracking.

Ignored and untracked local-only files remain in the visible local folder.

## Reserved local paths

These are control/state items and are not synchronized:

- `.sync_github_state`
- `.agents`
- `.codex`
- `.git`
- `sync_github.py`
- `test_sync_github.py`
- `__pycache__`
- `config.json`
- `README.md`

The program stops if the configured remote folder already contains any of these names at its top level.

## Changing the mapping

After the first run, `.sync_github_state/metadata.json` records the repository, branch, and remote subfolder. If those values later differ from `config.json`, the program stops.

To intentionally establish a new mapping, preserve local work, delete `.sync_github_state`, edit `config.json`, and run the program again.

## Limitations

- Primarily designed for Windows.
- HTTPS GitHub repository URLs only.
- Symbolic links are rejected.
- One visible local folder maps to one monorepo subfolder.
- Full private-GitHub operation requires Git Credential Manager and actual repository access on the Windows computer.


## Authentication progress and timeouts

Version 1.1 first checks stored credentials without allowing an invisible prompt. If authentication is required, it explicitly announces that a GitHub sign-in window or browser should open and runs the normal Git operation interactively. While waiting, it prints a status line every 15 seconds.

- `network_timeout_seconds` limits noninteractive GitHub network checks.
- `authentication_timeout_seconds` limits the interactive browser-login period.

This avoids a blank-looking terminal that waits indefinitely inside a hidden credential prompt.
