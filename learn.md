# Learning Plan: Ansible Vault for SSH Keys (Local → GitHub Actions)

**Goal:** Store your SSH private key in an Ansible Vault-encrypted file, use it from a playbook to connect to a remote host, rotate/update the key later, and run the same flow in GitHub Actions using a GitHub-hosted runner.

**Prerequisites:**
- Ansible installed (`ansible --version`)
- A GitHub account with a repository
- A target server reachable via SSH (or `localhost` for the first tests)
- Your existing SSH key pair on your laptop (`~/.ssh/id_ed25519` or similar)

---

## Phase 0 — Project Foundation (do once)

Before touching vault, create the folder structure that keeps secrets out of git.

| Step | Action | Command / Content |
|------|--------|-------------------|
| 0.1 | Create project folder | `mkdir ansible_vault_learning && cd ansible_vault_learning` |
| 0.2 | Initialize git | `git init` |
| 0.3 | Create `.gitignore` | See content below |
| 0.4 | Create `ansible.cfg` | Points at local inventory and `.vault_pass` |
| 0.5 | Create `inventory/hosts.yml` | Targets `localhost` first |
| 0.6 | Verify ansible reads config | `ansible-config dump --only-changed` |

### `.gitignore`
```gitignore
.vault_pass
*.retry
group_vars/all/vars.yml
__pycache__/
*.pyc
```

### `ansible.cfg`
```ini
[defaults]
inventory = inventory/hosts.yml
vault_password_file = .vault_pass
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = False
```

### `inventory/hosts.yml`
```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
```

**Success criteria:** Running `ansible-config dump --only-changed` shows your `inventory`, `vault_password_file`, and `host_key_checking` settings.

---

## Phase 1 — Vault Fundamentals (no SSH key yet)

Learn the vault command family with a dummy file so you don't risk your real key.

| Step | Action | Command |
|------|--------|---------|
| 1.1 | Create a vault password file | `printf '%s' "learn-vault-123" > .vault_pass && chmod 600 .vault_pass` |
| 1.2 | Create a dummy vars file | `mkdir -p group_vars/all` then write `group_vars/all/vars.yml` with `foo: bar` |
| 1.3 | Encrypt it | `ansible-vault encrypt group_vars/all/vars.yml` |
| 1.4 | View encrypted contents | `ansible-vault view group_vars/all/vars.yml` |
| 1.5 | Edit encrypted contents | `ansible-vault edit group_vars/all/vars.yml` (change `foo: bar` to `foo: baz`) |
| 1.6 | Change vault password | `ansible-vault rekey group_vars/all/vars.yml` (enter old, then new password) |
| 1.7 | Decrypt permanently (learn what it does) | `ansible-vault decrypt group_vars/all/vars.yml` — then re-encrypt immediately for safety |

### Key commands to memorize
- `ansible-vault create <file>` — new encrypted file
- `ansible-vault encrypt <file>` — encrypt existing plaintext file
- `ansible-vault decrypt <file>` — permanently decrypt
- `ansible-vault view <file>` — read-only peek
- `ansible-vault edit <file>` — decrypt → editor → re-encrypt
- `ansible-vault rekey <file>` — change vault password

**Success criteria:** You can encrypt, view, edit, rekey, and decrypt without errors. The encrypted file on disk starts with `$ANSIBLE_VAULT;`.

---

## Phase 2 — Store Your SSH Key in Vault

Now replace the dummy file with your real SSH key.

| Step | Action | Details |
|------|--------|---------|
| 2.1 | Create plaintext vars file | `group_vars/all/vars.yml` with your key as a YAML block scalar |
| 2.2 | Encrypt it | `ansible-vault encrypt group_vars/all/vars.yml` |
| 2.3 | Commit encrypted file to git | `git add group_vars/all/vault.yml` (after rename/encrypt) |
| 2.4 | Verify `.vault_pass` is NOT committed | `git status` should show nothing for `.vault_pass` |

### `group_vars/all/vars.yml` (plaintext, BEFORE encryption)
```yaml
# DO NOT COMMIT THIS UNENCRYPTED
ssh_private_key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAE...
  -----END OPENSSH PRIVATE KEY-----

ssh_public_key: |
  ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... alin@laptop

target_host: "203.0.113.10"
target_user: "alin"
```

### How to encrypt
```bash
ansible-vault encrypt group_vars/all/vars.yml
```
Then rename to `vault.yml` if you want convention:
```bash
mv group_vars/all/vars.yml group_vars/all/vault.yml
```
But note: Ansible automatically loads all `.yml` files in `group_vars/all/`, so the filename doesn't matter much as long as it's encrypted.

**Success criteria:** `group_vars/all/vault.yml` exists and starts with `$ANSIBLE_VAULT;`. Running `ansible-vault view group_vars/all/vault.yml` shows your key.

---

## Phase 3 — Use the Vaulted SSH Key in a Playbook

Write a playbook that reads the key from vault and writes it to a remote host (or localhost for testing).

### `playbook.yml`
```yaml
---
- name: Deploy SSH key from Ansible Vault
  hosts: all
  gather_facts: false

  tasks:
    - name: Ensure .ssh directory exists on target
      ansible.builtin.file:
        path: "{{ ansible_env.HOME }}/.ssh"
        state: directory
        mode: '0700'

    - name: Install vaulted private key
      ansible.builtin.copy:
        content: "{{ ssh_private_key }}"
        dest: "{{ ansible_env.HOME }}/.ssh/id_ansible_vault"
        mode: '0600'
        owner: "{{ ansible_user }}"

    - name: Install matching public key
      ansible.builtin.copy:
        content: "{{ ssh_public_key }}"
        dest: "{{ ansible_env.HOME }}/.ssh/id_ansible_vault.pub"
        mode: '0644'

    - name: Test SSH connectivity using vaulted key
      ansible.builtin.command:
        cmd: "ssh -i {{ ansible_env.HOME }}/.ssh/id_ansible_vault -o StrictHostKeyChecking=no {{ target_user }}@{{ target_host }} 'echo vault-key-works'"
      register: ssh_test
      changed_when: false
      failed_when: false

    - name: Show SSH test result (never print the key)
      ansible.builtin.debug:
        msg: "SSH test: {{ ssh_test.stdout | default('failed or skipped') }}"
```

### Running locally (against localhost)
```bash
ansible-playbook playbook.yml
```

### Running against a real host
```bash
ansible-playbook -l myserver playbook.yml
```

### Security rule
Never use `ansible.builtin.debug` with `msg: "{{ ssh_private_key }}"`. CI logs are visible to anyone with repo access.

**Success criteria:** The playbook runs, creates `~/.ssh/id_ansible_vault` with mode `600`, and the SSH test returns success (or at least attempts the connection).

---

## Phase 4 — Update / Rotate the SSH Key

This is the core of your request: "I want to update for example in a workflow with a new one and store in that vault."

### Scenario A: Rotate the key content (same vault password)

| Step | Action | Command |
|------|--------|---------|
| 4.1 | Generate new key pair locally | `ssh-keygen -t ed25519 -f ~/.ssh/id_ansible_new -N ""` |
| 4.2 | Edit the vaulted file | `ansible-vault edit group_vars/all/vault.yml` |
| 4.3 | Replace old key with new key | Paste new private/public key, save, exit editor |
| 4.4 | Re-run playbook | `ansible-playbook playbook.yml` — the `copy` module overwrites the old key file on the target |
| 4.5 | Authorize new public key on target | Append `id_ansible_new.pub` to `~/.ssh/authorized_keys` on the remote host |

### Scenario B: Change the vault password itself

| Step | Action | Command |
|------|--------|---------|
| 4.6 | Rekey the vault file | `ansible-vault rekey group_vars/all/vault.yml` |
| 4.7 | Update local `.vault_pass` | `printf '%s' "new-vault-password" > .vault_pass && chmod 600 .vault_pass` |
| 4.8 | Update GitHub secret | Set `ANSIBLE_VAULT_PASSWORD` to the new password in repo settings |

### Idempotency note
The `copy` module only changes the file if the content differs. Re-running the playbook with the same key does nothing. With a new key, it overwrites automatically.

**Success criteria:** After editing the vault and re-running, the target host has the new key and can log in with it.

---

## Phase 5 — GitHub Actions (GitHub-hosted runner)

This is where the vault connects to a workflow. The runner is a temporary Ubuntu VM; it does not have your key. It gets the vault password from a GitHub secret, decrypts the vault, and runs the playbook.

### 5.1 Add GitHub repository secrets

Repo → Settings → Secrets and variables → Actions → New repository secret:

| Secret name | Value |
|-------------|-------|
| `ANSIBLE_VAULT_PASSWORD` | Your vault password (e.g. `learn-vault-123` or whatever you set in Phase 1/4) |
| `SSH_PRIVATE_KEY` | Optional: contents of your private key, if you want the workflow to write it directly instead of pulling from vault |
| `TARGET_HOST` | Your server IP |
| `TARGET_USER` | SSH username on server |

### 5.2 Workflow file: `.github/workflows/deploy.yml`

```yaml
name: Deploy SSH Key from Vault

on:
  push:
    branches: ["main"]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest   # GitHub-hosted runner
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.x"

      - name: Install Ansible
        run: pip install ansible

      - name: Write vault password file from secret
        run: printf '%s' "${VAULT_PASSWORD}" > .vault_pass
        env:
          VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}

      - name: Write SSH private key from secret (optional alternative to vault)
        run: |
          mkdir -p ~/.ssh
          echo "${SSH_PRIVATE_KEY}" > ~/.ssh/id_github_actions
          chmod 600 ~/.ssh/id_github_actions
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Run playbook using vaulted key
        run: ansible-playbook -l myserver playbook.yml
        env:
          TARGET_HOST: ${{ secrets.TARGET_HOST }}
          TARGET_USER: ${{ secrets.TARGET_USER }}
```

> Note: `group_vars/all/vault.yml` is committed to the repo and is encrypted. The runner downloads it, decrypts it using `.vault_pass` (from the secret), and uses `ssh_private_key` inside the playbook.

### 5.3 How the runner connects to your server

Since the runner is GitHub-hosted:
- Your server must have a public IP and port 22 open.
- The runner SSHes **out** to your server using the private key from the vault.
- Add `ssh-keyscan` if you want to avoid host-key prompts on first connect:

```yaml
      - name: Pre-seed known_hosts
        run: ssh-keyscan -H ${{ secrets.TARGET_HOST }} >> ~/.ssh/known_hosts
```

**Success criteria:** You can trigger the workflow from the GitHub Actions tab ("Run workflow") and see the playbook execute against your remote server.

---

## Phase 6 — Advanced / Optional (learn after basics work)

| Topic | Why learn it | Example |
|-------|--------------|---------|
| Vault IDs | Use different passwords for different environments (dev vs prod) | `ansible-vault encrypt --vault-id dev@prompt group_vars/dev/vault.yml` |
| Multiple vault files | Separate `vault.yml` per environment | `group_vars/production/vault.yml`, `group_vars/staging/vault.yml` |
| `ansible-pull` | The target server pulls the repo and runs Ansible locally instead of being pushed to | Good if your server has no public IP |
| Ansible Vault with GitHub Actions + self-hosted runner | Runner lives on your server; no inbound SSH from internet needed | Install runner app on your server |

---

## Full Command Cheat Sheet (copy-paste reference)

```bash
# Setup
printf '%s' "mypassword" > .vault_pass && chmod 600 .vault_pass

# Vault lifecycle
ansible-vault create group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vars.yml
ansible-vault decrypt group_vars/all/vault.yml
ansible-vault view group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
ansible-vault rekey group_vars/all/vault.yml

# Run playbook
ansible-playbook playbook.yml
ansible-playbook --vault-password-file .vault_pass playbook.yml
ansible-playbook --ask-vault-pass playbook.yml
ansible-playbook -l myserver playbook.yml
ansible-playbook --syntax-check playbook.yml

# Inspect config
ansible-config dump --only-changed
ansible-config view
```

---

## Verification Checklist (tick off as you go)

- [ ] `.gitignore` blocks `.vault_pass` and unencrypted `vars.yml`
- [ ] `ansible.cfg` exists and `ansible-config dump` shows it
- [ ] `ansible-vault encrypt` works on a test file
- [ ] `ansible-vault view` shows plaintext after entering password
- [ ] `ansible-vault edit` saves changes and re-encrypts
- [ ] `ansible-vault rekey` changes password successfully
- [ ] Real SSH key is inside `group_vars/all/vault.yml` (encrypted)
- [ ] Playbook copies key to target with mode `0600`
- [ ] Playbook never prints `{{ ssh_private_key }}`
- [ ] GitHub secret `ANSIBLE_VAULT_PASSWORD` added
- [ ] Workflow triggers and runs `ansible-playbook` successfully
- [ ] Key rotation: edit vault → re-run playbook → new key works on target

---

## What NOT to Do (common mistakes)

- ❌ Commit `.vault_pass` to git
- ❌ Commit unencrypted `group_vars/all/vars.yml` containing a real key
- ❌ `debug: msg: "{{ ssh_private_key }}"` — leaks key in CI logs
- ❌ Use `ansible_ssh_pass` (plaintext password) in inventory
- ❌ Store the vault password inside the playbook or workflow YAML
