# 02 - Dev tools

Dev tools and git configuration for a remote Linux server: repo manager (ghq), GitHub CLI, Node.js (via nvm), Claude Code, SSH key auth, and commit signing.

Assumes [01-shell.md](01-shell.md) is already completed (`~/.local/bin` in PATH, zsh as default shell).

## Placeholders

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `<USER>` | Your Linux username | `jane` |
| `<EMAIL>` | Email associated with your GitHub account | `jane@example.com` |
| `<MACHINE_NAME>` | SSH key name, conventionally the machine hostname in uppercase | `MYSERVER` |

---

## 2.1 ghq - repo manager

ghq organizes git repositories into a predictable directory tree (`~/code/github.com/<org>/<repo>`). It is a static Go binary with no runtime dependencies.

**Precondition:** `~/.local/bin` exists and is in PATH (from runbook 01).

**Action:**

```bash
GHQ_VERSION=1.9.4   # check latest: https://github.com/x-motemen/ghq/releases
# set ARCH to match your machine: amd64 for x86_64, arm64 for ARM
ARCH=amd64
curl -sL "https://github.com/x-motemen/ghq/releases/download/v${GHQ_VERSION}/ghq_linux_${ARCH}.zip" -o /tmp/ghq.zip
sudo apt-get install -y unzip
unzip -q /tmp/ghq.zip -d /tmp/ghq
mkdir -p ~/.local/bin
mv /tmp/ghq/ghq_linux_${ARCH}/ghq ~/.local/bin/
rm -rf /tmp/ghq /tmp/ghq.zip
```

**Verify:**

```bash
ghq --version
# expected: ghq version 1.9.4 (or the version you installed)
```

---

## 2.2 gh CLI - GitHub from the terminal

**Action:**

```bash
sudo apt-get install -y gh
```

**Verify:**

```bash
gh --version
# expected: gh version X.Y.Z
```

---

## 2.3 nvm + Node.js

nvm installs Node.js in `~/.nvm` without touching system directories. Required for Claude Code.

**Precondition:** zsh is the default shell and `.zshrc` already sources nvm (from runbook 01).

**Action:**

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

The installer appends lines to `.bashrc`. These are redundant if your `.zshrc` already has the nvm block from runbook 01 - leave them, they cause no harm.

Reload the shell, then install Node.js:

```bash
exec zsh
nvm install --lts
nvm use --lts
```

**Verify:**

```bash
node --version
# expected: vXX.Y.Z (LTS version)
npm --version
# expected: X.Y.Z
```

---

## 2.4 Claude Code

**Precondition:** Node.js and npm installed (step 2.3).

**Action:**

```bash
npm install -g @anthropic-ai/claude-code
```

**Verify:**

```bash
claude --version
# expected: X.Y.Z
```

---

## 2.5 Git globals

Set identity, default branch, and ghq root directory.

**Action:**

```bash
git config --global user.name '<USER>'
git config --global user.email '<EMAIL>'
git config --global init.defaultBranch main
git config --global ghq.root '~/code'
mkdir -p ~/code
```

**Verify:**

```bash
git config --global --list | grep -E 'user\.|init\.|ghq\.'
# expected: user.name, user.email, init.defaultbranch=main, ghq.root=~/code
```

---

## 2.6 SSH key for GitHub

Convention: key name matches the machine hostname, uppercase.

**Action:**

Generate an ed25519 key:

```bash
ssh-keygen -t ed25519 -C "<USER>@<MACHINE_NAME>" -f ~/.ssh/<MACHINE_NAME> -N ''
cat ~/.ssh/<MACHINE_NAME>.pub
```

On GitHub (Settings > SSH and GPG keys > New SSH key), register the public key **twice**:

1. Type **Authentication key** - for push/pull
2. Type **Signing key** - for commit signing

Both entries use the same public key content.

Add to `~/.ssh/config`:

```
Host github.com
  IdentityFile ~/.ssh/<MACHINE_NAME>
  User git
```

**Verify:**

```bash
ssh-keyscan github.com >> ~/.ssh/known_hosts 2>/dev/null
ssh -T git@github.com
# expected: Hi <github-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

---

## 2.7 gh CLI authentication - device flow (headless)

The device flow works without a browser on the remote machine: it gives you a code to enter on any browser.

**Precondition:** SSH key registered on GitHub (step 2.6), `~/.ssh/config` has the github.com entry.

**Action:**

```bash
gh auth login --hostname github.com --git-protocol ssh --web
# select SSH when asked for protocol
# select "Login with a web browser" when asked for method
# open https://github.com/login/device on any browser, enter the code shown
```

gh automatically adds the SSH key to GitHub as an authentication key during login.

**Verify:**

```bash
gh ssh-key list
# expected: your <MACHINE_NAME> key appears in the list

ssh -T git@github.com
# expected: Hi <github-username>! You've successfully authenticated...
```

---

## 2.8 Commit signing with SSH

Configure git to sign commits with the same SSH key used for authentication.

**Precondition:** SSH key registered on GitHub as a **signing key** (step 2.6).

**Action:**

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/<MACHINE_NAME>.pub
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

**Verify:**

```bash
# run inside any git repo
git commit --allow-empty -m "chore: verify commit signing"
git log --show-signature -1
# expected: "Good ssh signature" (requires allowed_signers, see below)
```

---

## 2.9 Local signature verification (optional)

Without this, `git log --show-signature` shows an error instead of "Good ssh signature". Not required for GitHub to show commits as "Verified", but useful for local verification.

**Action:**

```bash
echo "<EMAIL> $(cat ~/.ssh/<MACHINE_NAME>.pub)" > ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

**Verify:**

```bash
git log --show-signature -1
# expected: Good "git" signature for <EMAIL> with ED25519 key SHA256:...
```

---

## Common issues on headless servers

| Problem | Cause | Fix |
|---------|-------|-----|
| `gh auth login` asks for a browser | Default auth flow needs a GUI browser | Use `--web` for device flow, or `--with-token < token.txt` |
| `gh auth refresh -s <scope>` asks for a browser | Scope addition triggers browser flow | Add all needed scopes during the initial `gh auth login` |
| Commits show "Unverified" on GitHub | Key registered only as authentication, not signing | Add the same key as a **Signing key** on GitHub |
| `git log --show-signature` errors | `gpg.ssh.allowedSignersFile` not set | Run step 2.9 |
| Branch protection blocks merge | "Require signed commits" is enabled and source commits are unsigned | Configure signing (step 2.8) before the first commit; fixing unsigned commits later requires interactive rebase |

> Configure signing before your first commit in any project. Retroactively signing existing commits requires rebase.
