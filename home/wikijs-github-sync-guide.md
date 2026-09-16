---
title: wikijs-github-sync-guide
description: 
published: true
date: 2026-09-16T20:42:48.677Z
tags: 
editor: markdown
dateCreated: 2026-09-16T15:52:24.606Z
---

# Guide: Syncing Wiki.js with GitHub (Git-backed Storage)

**Objective:** Configure Wiki.js to use a GitHub repository as its content storage (instead of relying solely on PostgreSQL), enabling versioning, commit-based history, and portable markdown content.

**Document status:** POC in progress. This document records the full process, including what worked, the SSH errors encountered, and the current attempt to resolve it via HTTPS.

---

## Prerequisites

- Wiki.js 2.5.314 running locally (Node.js + PostgreSQL, see separate installation guide)
- Git installed on the system (verified with `git --version`) — required because Wiki.js invokes `git` as a subprocess
- GitHub account with permissions to create repositories and deploy keys

---

## Part 1: Environment setup (worked regardless of transport)

These steps succeeded on their own and remain valid no matter which authentication method (SSH or HTTPS) ends up being used.

### 1.1 Create a dedicated GitHub repository

A new, empty repository was created, dedicated exclusively to Wiki.js (a subfolder of an existing repo or submodules cannot be used):

```
reneargueta/WikiRW
```

### 1.2 Generate the SSH key pair

```powershell
ssh-keygen -t ed25519 -C "wikijs@rulesware" -f C:\Users\rene.argueta\Documents\wikijs_ssh_key -N ""
```

- `-t ed25519`: key type (modern, secure)
- `-C`: comment/label to identify the key
- `-f`: path where the key pair is saved (`wikijs_ssh_key` private, `wikijs_ssh_key.pub` public)
- `-N ""`: no passphrase (required because Wiki.js can't respond to an interactive prompt)

### 1.3 Add the deploy key to GitHub

1. Repo → **Settings → Deploy keys → Add deploy key**
2. Descriptive title (e.g. `wikijs-sync`)
3. Paste the contents of `wikijs_ssh_key.pub`
4. **Check "Allow write access"** (critical — without this, Wiki.js can read but never push changes back)

---

## Part 2: SSH connection attempt (ultimately abandoned)

The steps below confirmed the SSH key itself was valid and correctly authorized on GitHub's side — but the SSH connection *from Wiki.js specifically* never succeeded (see Parts 3 and 4). SSH was eventually abandoned in favor of HTTPS.

### 2.1 Confirm the key works manually (outside of Wiki.js)

Before trusting Wiki.js's own configuration, the key was validated directly:

```powershell
git -c core.sshCommand="ssh -i C:\\Users\\rene.argueta\\Documents\\wikijs_ssh_key" push -u origin main
```

**Important note:** in PowerShell, paths inside `core.sshCommand` needed **double backslashes** (`\\`) — with a single backslash, the internal SSH parser swallowed the slashes and the path arrived corrupted (`C:Usersrene.argueta...`).

This manual push worked and created the `main` branch on the remote repo, confirming the key and deploy key were correctly set up on GitHub's side.

### 2.2 Field configuration in Wiki.js (Administration → Storage → Git)

| Field | Value |
|---|---|
| Authentication Type | `ssh` |
| Repository URI | `git@github.com:reneargueta/WikiRW.git` |
| Branch | `main` |
| SSH Private Key Mode | `contents` |
| Default Author Email / Name | Rene's email and name |
| Local Repository Path | `./data/repo` (default) |
| Sync Direction | Bi-directional |
| Verify SSL Certificate | On |

---

## Part 3: Errors encountered

### Error 1 — `error in libcrypto` when pasting the private key

**Full message:**
```
Fetching origin Load key "...\data\secure\git-ssh.pem": error in libcrypto
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

**Root cause:** when copying the private key file's content from Notepad and pasting it into the **B - SSH Private Key Contents** field, Windows-style line endings (CRLF, `\r\n`) slipped into the pasted text. The OpenSSL/libcrypto parser Wiki.js uses internally is strict about PEM formatting and rejects the key if it has extra `\r` characters — even though the pasted text looks visually identical to the original.

**How it was diagnosed:** the key file was read from PowerShell (stripped of CRLF) and copied straight to the clipboard, bypassing Notepad entirely:
```powershell
(Get-Content -Raw C:\Users\rene.argueta\Documents\wikijs_ssh_key) -replace "`r`n", "`n" | Set-Clipboard
```

### Error 2 — The SSH process hangs indefinitely

After resolving Error 1, Wiki.js's Status still showed red, but this time with no specific error message — just "An unexpected error occurred" when clicking Apply, and in the server logs, a multi-minute gap between "Fetch updates from remote..." and the next line.

**Diagnosis:** the exact key file Wiki.js had generated on disk (`data\secure\git-ssh.pem`) was tested directly from PowerShell:
```powershell
ssh -i "C:\Users\rene.argueta\Documents\wiki-2.5.314\data\secure\git-ssh.pem" -T git@github.com
```
Result: `Hi reneargueta/WikiRW! You've successfully authenticated, but GitHub does not provide shell access.` — meaning **the key itself was perfectly fine**. The problem wasn't the key; it was that Wiki.js's internal SSH subprocess didn't have `github.com`'s host fingerprint registered in `known_hosts`, and with no interactive terminal available to confirm "do you trust this host?", the process sat waiting for that response indefinitely instead of failing fast.

### Error 3 — `ssh-keyscan` doesn't support GitHub's key exchange method

When attempting to register GitHub's fingerprint manually:
```powershell
ssh-keyscan github.com >> $env:USERPROFILE\.ssh\known_hosts
```
The result was only repeated error lines:
```
choose_kex: unsupported KEX method sntrup761x25519-sha512@openssh.com
```
**Cause:** the version of `ssh-keyscan` bundled with Windows is older and doesn't support the post-quantum algorithm GitHub advertises first during negotiation.

**Fix that worked for this specific step:**
```powershell
ssh-keyscan -t rsa,ecdsa,ed25519 github.com >> $env:USERPROFILE\.ssh\known_hosts
```
This forced classic, compatible key types and did return GitHub's real fingerprints.

---

## Part 4: Attempts that failed

### Failed attempt 1 — `GIT_SSH_COMMAND` environment variable

```powershell
$env:GIT_SSH_COMMAND = "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL"
node server
```

**Result:** no effect — Wiki.js's Status kept hanging exactly the same way.

**Why it didn't work:** the logs showed Wiki.js building its own SSH command internally (log line "Setting SSH Command config..."), likely using `core.sshCommand` at the Git repository config level — which takes precedence over the parent process's environment variable. The environment variable never actually reached the real subprocess.

### Failed attempt 2 — Updated `~/.ssh/known_hosts` via `ssh-keyscan`

Even though the Windows user profile's `known_hosts` ended up with GitHub's correct fingerprints (confirmed with `Get-Content`), and even though a manual SSH connection worked perfectly using that same file, **Wiki.js kept hanging in exactly the same way** after clicking Apply following this change.

**Unconfirmed hypothesis:** it's possible Wiki.js doesn't inherit the Windows user profile's `known_hosts` the same way an interactive PowerShell session does, depending on how Node.js invokes the underlying `git`/`ssh` subprocess (possibly a difference in user/environment context for the process).

### Attempt not completed — `~/.ssh/config` file with `StrictHostKeyChecking no`

It was proposed to create/edit:
```
Host github.com
    StrictHostKeyChecking no
    UserKnownHostsFile NUL
```
in `$env:USERPROFILE\.ssh\config`, under the hypothesis that the actual SSH client (`ssh.exe`) respects this file regardless of which process invokes it. **This attempt was not tested/confirmed** before moving on to the next strategy (HTTPS).

---

## Part 5: Current strategy — Switching from SSH to HTTPS

Since the failure pattern consistently pointed to the SSH host-verification layer (something specific to how Wiki.js handles `known_hosts` internally on Windows), it was decided to bypass SSH entirely and use HTTPS authentication with a **Fine-grained Personal Access Token (PAT)**, since HTTPS relies on standard TLS certificates rather than manual SSH fingerprints.

### Steps:

1. **Generate the token on GitHub:**
   - Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token
   - Repository access: Only select repositories → `WikiRW`
   - Permissions → Repository permissions → **Contents: Read and write**
   - Copy the generated token (`github_pat_...`) — it's only shown once

2. **Reconfigure Wiki.js:**
   - Authentication Type: `basic`
   - Repository URI: `https://github.com/reneargueta/WikiRW.git`
   - Username: `reneargueta`
   - Password / PAT: the generated token
   - The SSH configuration fields go unused (they're ignored when set to `basic`)

3. **Apply Changes** and check the Status panel / server logs.

**Result:** pending confirmation at the time this document was written.

---

## Lessons learned / notes for the future

- **Copying private keys from Windows is error-prone.** Using PowerShell to explicitly replace `\r\n` with `\n` is more reliable than copying from Notepad.
- **Validating the key independently (outside of the tool that will use it) saves diagnostic time** — manually testing `ssh -i <path> -T git@github.com` quickly isolated that the problem wasn't the key, but the host-key-checking layer.
- **Environment variables don't always propagate to subprocesses** when the application (in this case Wiki.js) explicitly builds its own Git command/configuration — in those cases, the global config file (`~/.ssh/config`) is more reliable than one-off environment variables.
- **HTTPS + a Fine-grained PAT is a simpler alternative** when SSH introduces infrastructure friction (host keys, corporate port-22 policies, etc.), at the cost of managing token expiration periodically.
