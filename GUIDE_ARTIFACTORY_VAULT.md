# Ansible Vault — JFrog Artifactory API Key Guideline

**Audience:** DevOps engineers, platform teams, SREs  
**Author:** Tech Lead DevOps  
**Status:** Production-ready starter pattern  
**Last updated:** Oct 2026

---

## 1. Executive Summary (read this first)

You have a JFrog Artifactory API key. It is a secret. It must never live in plain text in git, CI logs, or chat.

This guideline shows you how to:

1. Store the API key in **Ansible Vault** (encrypted at rest).
2. Use it from an **Ansible playbook** to talk to Artifactory.
3. Run the same playbook in **GitHub Actions** using a GitHub-hosted runner.
4. **Rotate** the key when it expires or is compromised.

**Golden rules (memorize these):**

| Rule | Why |
|------|-----|
| Encrypted vault files are safe to commit to git | They are useless without the vault password |
| `.vault_pass` and unencrypted vars are **never** committed | They unlock the secrets |
| Never `debug` or `print` the API key | CI logs are readable by anyone with repo access |
| Every task that touches the key uses `no_log: true` | Prevents accidental secret leakage in output |
| Rotate the key by editing the vault, not by rewriting playbooks | One source of truth |

---

## 2. Architecture (how it fits together)

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────────┐
│   GitHub Repo   │      │  GitHub Actions  │      │   JFrog Artifactory │
│                 │      │   (runner)       │      │                     │
│ playbook.yml    │─────▶│ ansible-playbook │─────▶│ https://.../artifactory │
│ vault.yml       │      │ + .vault_pass    │      │ API key: Bearer <key> │
│ (encrypted)     │      │ (from secret)    │      │                     │
└─────────────────┘      └──────────────────┘      └─────────────────────┘
         ▲
         │
┌─────────────────┐
│   Your laptop   │
│ ansible-vault   │
│ edit/view       │
└─────────────────┘
```

**Data flow:**
- **You** create/edit the key locally with `ansible-vault edit`.
- **Git** stores only the encrypted `vault.yml`.
- **GitHub runner** receives the vault password via a secret, writes `.vault_pass`, and runs the playbook.
- **Playbook** reads `artifactory_api_key` from vault and sends it to Artifactory in an `Authorization: Bearer` header.

---

## 3. Repository layout

```text
ansible_learning/
├── ansible.cfg                 # Ansible defaults (inventory, vault password file)
├── .gitignore                  # Blocks .vault_pass and unencrypted vars
├── GUIDE.md                    # General Ansible Vault starter guide
├── GUIDE_ARTIFACTORY_VAULT.md  # This document (JFrog scenario)
├── learn.md                    # Learning plan
├── inventory/
│   └── hosts.yml               # Targets (localhost for testing)
├── group_vars/
│   └── all/
│       ├── vars.yml.example    # Template: copy, fill, encrypt
│       └── vault.yml           # Encrypted: committed to git
├── playbook.yml                # Playbook that calls JFrog Artifactory
└── .github/
    └── workflows/
        └── artifactory.yml     # CI workflow
```

---

## 4. Phase 0 — Project setup (do once)

Create the skeleton so Ansible knows where to find things.

### 4.1 `ansible.cfg`

```ini
[defaults]
inventory = inventory/hosts.yml
vault_password_file = .vault_pass
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = False
```

### 4.2 `inventory/hosts.yml`

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
```

### 4.3 `.gitignore`

```gitignore
.vault_pass
*.retry
group_vars/all/vars.yml
__pycache__/
*.pyc
```

### 4.4 Verify Ansible reads your config

```bash
ansible-config dump --only-changed
```

You should see `inventory`, `vault_password_file`, and `host_key_checking` listed.

---

## 5. Phase 1 — Get your JFrog Artifactory API key

1. Log in to JFrog Artifactory (or your company's instance).
2. Go to **User Profile → Edit Profile → API Key**.
3. Copy the API key. It looks something like:

   ```
   AKCp8jwmV5Qi4m8...
   ```

4. Treat it like a password. Do not email it, do not paste it in Slack, do not commit it.

---

## 6. Phase 2 — Store the key in Ansible Vault

### 6.1 Create the plaintext variables file

`group_vars/all/vars.yml`:

```yaml
# ---------------------------------------------------------------------------
# JFrog Artifactory credentials
# DO NOT COMMIT THIS FILE UNENCRYPTED
# ---------------------------------------------------------------------------

artifactory_url: "https://yourcompany.jfrog.io/artifactory"
artifactory_username: "alin"
artifactory_api_key: "AKCp8jwmV5...YOUR_REAL_KEY_HERE"
artifactory_repo: "generic-local"
```

### 6.2 Create the vault password file

```bash
printf '%s' "your-chosen-vault-password" > .vault_pass
chmod 600 .vault_pass
```

> `.vault_pass` is gitignored. Each developer and the CI runner creates their own copy with the same password.

### 6.3 Encrypt the file

```bash
ansible-vault encrypt group_vars/all/vars.yml
```

Ansible renames nothing; the file on disk now starts with:

```text
$ANSIBLE_VAULT;1.1;AES256
```

### 6.4 Commit the encrypted file

```bash
git add group_vars/all/vars.yml
git commit -m "Add encrypted JFrog Artifactory vault variables"
```

> Never commit `group_vars/all/vars.yml` (unencrypted). Never commit `.vault_pass`.

---

## 7. Phase 3 — Use the key in a playbook

`playbook.yml`:

```yaml
---
- name: JFrog Artifactory integration using Ansible Vault
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Fail early if API key is missing
      ansible.builtin.fail:
        msg: "artifactory_api_key is not defined. Encrypt group_vars/all/vars.yml first."
      when: artifactory_api_key is undefined

    - name: Ping Artifactory to verify connectivity
      ansible.builtin.uri:
        url: "{{ artifactory_url }}/api/system/ping"
        method: GET
        return_content: true
        validate_certs: true
        headers:
          Authorization: "Bearer {{ artifactory_api_key }}"
      register: artifactory_ping
      failed_when: false
      no_log: true   # hides request/response from logs

    - name: Report ping status (no secrets printed)
      ansible.builtin.debug:
        msg: "Artifactory ping status: {{ artifactory_ping.status | default('unknown') }}"

    - name: Upload a sample artifact
      ansible.builtin.uri:
        url: "{{ artifactory_url }}/{{ artifactory_repo }}/ansible-test.txt"
        method: PUT
        return_content: true
        headers:
          Authorization: "Bearer {{ artifactory_api_key }}"
          Content-Type: "text/plain"
        body: "Deployed by Ansible Vault"
        body_format: raw
      register: artifactory_upload
      no_log: true

    - name: Report upload status
      ansible.builtin.debug:
        msg: "Upload status: {{ artifactory_upload.status | default('unknown') }}"
```

### Key security settings explained

| Setting | Purpose |
|---------|---------|
| `no_log: true` | Stops Ansible from printing the task's request/response (which would contain the API key) in stdout and CI logs |
| `failed_when: false` on ping | Lets us report status gracefully instead of crashing the playbook |
| Never `debug: msg: "{{ artifactory_api_key }}"` | Would leak the secret to anyone reading the output |

### Run locally

```bash
ansible-playbook playbook.yml
```

Expected output (with a valid key):

```
TASK [Ping Artifactory to verify connectivity] ********************************
ok: [localhost]

TASK [Report ping status] *****************************************************
ok: [localhost] => {
    "msg": "Artifactory ping status: 200"
}
```

---

## 8. Phase 4 — GitHub Actions (GitHub-hosted runner)

The runner is a temporary Ubuntu VM. It does not have your key. It receives the vault password from a GitHub secret, decrypts the vault, and runs the playbook.

### 8.1 Add GitHub repository secrets

Go to **Settings → Secrets and variables → Actions → New repository secret**:

| Secret name | Value |
|-------------|-------|
| `ANSIBLE_VAULT_PASSWORD` | Your vault password (the one inside `.vault_pass`) |

### 8.2 Workflow file

`.github/workflows/artifactory.yml`:

```yaml
name: JFrog Artifactory Deploy

on:
  push:
    branches: ["main"]
  workflow_dispatch:

jobs:
  artifactory:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.x"

      - name: Install Ansible
        run: pip install ansible

      - name: Write vault password file from GitHub secret
        run: printf '%s' "${VAULT_PASSWORD}" > .vault_pass
        env:
          VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}

      - name: Run playbook against Artifactory
        run: ansible-playbook playbook.yml
```

### 8.3 How it works at runtime

1. GitHub spins up `ubuntu-latest`.
2. The step writes `.vault_pass` from the secret `ANSIBLE_VAULT_PASSWORD`.
3. `ansible-playbook` reads `.vault_pass`, decrypts `group_vars/all/vars.yml` (committed to git), and extracts `artifactory_api_key`.
4. The playbook sends `Authorization: Bearer <key>` to Artifactory.
5. The runner is destroyed when the job finishes. No secret persists.

---

## 9. Phase 5 — Rotate the API key (without breaking CI)

API keys expire or get compromised. Here is how to rotate.

### Scenario A: New JFrog API key, same vault password

```bash
# 1. Generate or copy your new Artifactory API key
# 2. Edit the encrypted vault file
ansible-vault edit group_vars/all/vars.yml

# 3. Replace the old artifactory_api_key value with the new one
# 4. Save and exit the editor (file is re-encrypted automatically)
# 5. Commit and push
git add group_vars/all/vars.yml
git commit -m "Rotate JFrog Artifactory API key"
git push
```

CI will pick up the new key on the next run. No changes to GitHub secrets needed.

### Scenario B: Change the vault password itself

```bash
# 1. Rekey the vault file
ansible-vault rekey group_vars/all/vars.yml

# 2. Update local .vault_pass
printf '%s' "new-vault-password" > .vault_pass
chmod 600 .vault_pass

# 3. Update GitHub secret
#    Settings → Secrets → ANSIBLE_VAULT_PASSWORD = new-vault-password

# 4. Commit nothing (vault file content unchanged, only password changed)
```

---

## 10. Security checklist (never skip)

| # | Check | Status |
|---|-------|--------|
| 1 | `group_vars/all/vault.yml` starts with `$ANSIBLE_VAULT;` | ☐ |
| 2 | `.vault_pass` is listed in `.gitignore` | ☐ |
| 3 | Unencrypted `group_vars/all/vars.yml` is **not** in git | ☐ |
| 4 | Every task using `artifactory_api_key` has `no_log: true` | ☐ |
| 5 | No `debug` task prints `{{ artifactory_api_key }}` | ☐ |
| 6 | `ANSIBLE_VAULT_PASSWORD` is a GitHub secret, not in workflow YAML | ☐ |
| 7 | You can run `ansible-vault view` and see your key | ☐ |

---

## 11. Troubleshooting

| Problem | Likely cause | Fix |
|---------|--------------|-----|
| `Vault password incorrect` | Wrong `.vault_pass` or wrong GitHub secret | Verify password matches the one used during `ansible-vault encrypt` |
| `artifactory_api_key is not defined` | Vault not loaded or file not encrypted properly | Run `ansible-vault view group_vars/all/vars.yml` to confirm contents |
| `401 Unauthorized` from Artifactory | API key expired or wrong scopes | Rotate the key in JFrog, update vault, re-run |
| `no_log: true` not hiding output | Task still prints in debug | Remove any `debug` tasks that reference the key or response body |

---

## 12. One-page cheat sheet

```bash
# Setup
printf '%s' "mypassword" > .vault_pass && chmod 600 .vault_pass

# Vault
ansible-vault encrypt group_vars/all/vars.yml
ansible-vault view group_vars/all/vars.yml
ansible-vault edit group_vars/all/vars.yml
ansible-vault rekey group_vars/all/vars.yml

# Run
ansible-playbook playbook.yml
ansible-playbook --vault-password-file .vault_pass playbook.yml

# Config check
ansible-config dump --only-changed
```

---

## 13. Remember this forever

> **Ansible Vault = encrypted file + password.**  
> The encrypted file goes in git. The password goes in `.vault_pass` (local) or a GitHub secret (CI).  
> The playbook reads the variable and sends it to the API.  
> `no_log: true` keeps it out of logs.  
> Rotate by editing the vault, not the playbook.

If you remember nothing else, remember: **secrets are encrypted at rest, never logged, and rotated through the vault.**
