# Ansible Vault — GitHub Tokens (Simple Starter Guide)

Manage GitHub personal access tokens (PATs) inside an **Ansible Vault**-encrypted file and use them from a playbook, locally and in GitHub Actions.

No roles, no collections, no galaxy — just one playbook, one encrypted vars file, and one password.

---

## 0. What you get in this repo

```
ansible_learning/
├── ansible.cfg                 # ansible defaults (inventory, vault password file)
├── .gitignore                  # keeps .vault_pass and unencrypted vars out of git
├── GUIDE.md                    # this file
├── inventory/hosts.yml         # run everything on localhost
├── group_vars/
│   └── all/
│       ├── vars.yml.example    # copy, fill in real values, then encrypt
│       └── vault.yml           # ENCRYPTED (created by you, see Step 4)
├── playbook.yml                # demo: loads token from vault, calls GitHub API
└── .github/workflows/ansible.yml  # GitHub Action that runs the playbook
```

---

## 1. Install Ansible (first time)

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y python3-pip
pip3 install --user ansible

# macOS
brew install ansible

# verify
ansible --version
```

Add `~/.local/bin` to your `PATH` if `ansible` is not found after pip install.

---

## 2. Create a GitHub token

1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens (recommended) or Classic tokens.
2. Give it the scopes your playbook needs (e.g. `repo`, `workflow`, `read:org`).
3. Copy the token — you only see it once.

---

## 3. Prepare your variables (unencrypted, local only)

```bash
cp group_vars/all/vars.yml.example group_vars/all/vars.yml
```

Edit `group_vars/all/vars.yml` and paste your real token:

```yaml
github_token: "ghp_YOUR_REAL_TOKEN_HERE"
github_owner: "your-github-username"
github_repo: "ansible_learning"
```

> ⚠️ This file contains a secret. It is listed in `.gitignore` — **do not** remove it from `.gitignore` and never commit it.

---

## 4. Encrypt the variables with Ansible Vault

This is the key step. `ansible-vault` encrypts the file with a password you choose.

```bash
# encrypt (you will be prompted for a new vault password)
ansible-vault encrypt group_vars/all/vars.yml
```

To inspect or change it later:

```bash
# view (enter vault password when prompted)
ansible-vault view group_vars/all/vars.yml

# edit (opens your $EDITOR, decrypts on the fly, re-encrypts on save)
ansible-vault edit group_vars/all/vars.yml

# change the vault password
ansible-vault rekey group_vars/all/vars.yml

# decrypt permanently (rarely needed)
ansible-vault decrypt group_vars/all/vars.yml
```

The encrypted file now looks like:

```
$ANSIBLE_VAULT;1.1;AES256
35383437...
```

It is safe to commit to git.

---

## 5. Create the vault password file (so you don't type the password every run)

Create `.vault_pass` in the project root containing **only** your vault password:

```bash
printf '%s' "your-chosen-vault-password" > .vault_pass
chmod 600 .vault_pass
```

`ansible.cfg` already points at it:

```ini
[defaults]
vault_password_file = .vault_pass
```

`.vault_pass` is in `.gitignore` — never commit it.  
Each developer/CI runner creates their own `.vault_pass` with the same vault password.

> If you rekey the vault (change its password), update `.vault_pass` everywhere too.

---

## 6. Run the playbook locally

```bash
ansible-playbook playbook.yml
```

Expected output (with a fake/demo token the API call returns 401, which is fine — the point is the token was loaded from the encrypted vault and **never printed**):

```
TASK [Prove the vaulted token is available] ********************************
ok: [localhost] => {
    "msg": "GitHub token loaded from vault: ***HIDDEN*** (length 40)"
}

TASK [Call GitHub API to verify the token works] ***************************
ok: [localhost]

TASK [Print authenticated GitHub login] ************************************
ok: [localhost] => {
    "msg": "Authenticated as: auth failed (status 401)"
}
```

With a real token, the last task prints `Authenticated as: <your-github-login>`.

---

## 7. Use it in GitHub Actions

You need to get the vault password into the runner **without** writing it to the workflow file. Store it as a repository secret:

1. In GitHub: **Settings → Secrets and variables → Actions → New repository secret**.
2. Name: `ANSIBLE_VAULT_PASSWORD`
3. Value: the same vault password you used in `ansible-vault encrypt` / `.vault_pass`.

The workflow (`.github/workflows/ansible.yml`) writes that secret to `.vault_pass` at runtime and runs the playbook:

```yaml
- name: Write vault password file from GitHub secret
  run: printf '%s' "${VAULT_PASSWORD}" > .vault_pass
  env:
    VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}

- name: Run playbook (uses encrypted vault)
  run: ansible-playbook playbook.yml
```

The encrypted `group_vars/all/vault.yml` is committed to the repo; only the password is a secret. That is the whole point of Ansible Vault.

Trigger the run manually from the **Actions** tab (`workflow_dispatch`) or on push to `main`.

---

## 8. Command cheat sheet

| Goal | Command |
|------|---------|
| Create / encrypt the vault file | `ansible-vault encrypt group_vars/all/vault.yml` |
| View encrypted contents | `ansible-vault view group_vars/all/vault.yml` |
| Edit encrypted contents | `ansible-vault edit group_vars/all/vault.yml` |
| Change vault password | `ansible-vault rekey group_vars/all/vault.yml` |
| Decrypt permanently | `ansible-vault decrypt group_vars/all/vault.yml` |
| Run playbook locally | `ansible-playbook playbook.yml` |
| Run with explicit password file | `ansible-playbook --vault-password-file .vault_pass playbook.yml` |
| Syntax check | `ansible-playbook --syntax-check playbook.yml` |

---

## 9. Security reminders

- ✅ Commit `group_vars/all/vault.yml` (encrypted) — it is safe.
- ✅ Commit `.github/workflows/ansible.yml` and `playbook.yml` — they contain no secrets.
- ❌ Never commit `.vault_pass`.
- ❌ Never commit an unencrypted `group_vars/all/vars.yml`.
- ❌ Never print `{{ github_token }}` in a `debug` task (CI logs are visible).
- Rotate the GitHub token and rekey the vault periodically.

---

## TL;DR (30 seconds)

```bash
cp group_vars/all/vars.yml.example group_vars/all/vars.yml
# edit vars.yml with your real token, then:
ansible-vault encrypt group_vars/all/vars.yml
printf '%s' "<vault-password>" > .vault_pass && chmod 600 .vault_pass
ansible-playbook playbook.yml
```

Then add `ANSIBLE_VAULT_PASSWORD` as a GitHub repository secret and let `.github/workflows/ansible.yml` run it in CI.
