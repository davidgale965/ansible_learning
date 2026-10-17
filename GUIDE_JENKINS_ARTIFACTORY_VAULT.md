# Ansible Vault + Jenkins — Download Artifact from JFrog Artifactory

**Audience:** DevOps engineers, platform teams, SREs  
**Author:** Tech Lead DevOps  
**Status:** Production-ready starter pattern  
**Last updated:** Oct 2026

---

## 1. Executive Summary (read this first)

You need Jenkins to download an artifact from JFrog Artifactory. Artifactory requires an API key. That key is a secret.

This guideline shows you how to:

1. Store the Artifactory API key in **Ansible Vault** (encrypted at rest).
2. Use it from an **Ansible playbook** to download an artifact via the Artifactory REST API.
3. Run the playbook from **Jenkins** using a Jenkins credential for the vault password.
4. **Rotate** the key when it expires or is compromised.

**Golden rules (memorize these):**

| Rule | Why |
|------|-----|
| Encrypted vault files are safe to commit to git | They are useless without the vault password |
| `.vault_pass` and unencrypted vars are **never** committed | They unlock the secrets |
| Jenkins receives the vault password as a **credential**, not plain text | Protects the key in CI |
| Every task that touches the API key uses `no_log: true` | Prevents secret leakage in Jenkins console output |
| Never `debug` or `print` the API key | Jenkins build logs are readable by anyone with job access |
| Rotate the key by editing the vault, not by rewriting the playbook | One source of truth |

---

## 2. Architecture (how it fits together)

```
┌─────────────────┐      ┌──────────────┐      ┌─────────────────────┐
│   Git Repo      │      │   Jenkins    │      │   JFrog Artifactory │
│                 │      │  Controller  │      │                     │
│ playbook.yml    │─────▶│ ansible-play │─────▶│ https://.../artifactory │
│ vault.yml       │      │ book         │      │ API key: Bearer <key> │
│ (encrypted)     │      │ + vault pass │      │                     │
│                 │      │ (credential) │      │                     │
└─────────────────┘      └──────────────┘      └─────────────────────┘
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
- **Jenkins** checks out the repo, injects the vault password from a Jenkins credential, and runs `ansible-playbook`.
- **Playbook** reads `artifactory_api_key` from vault and sends `Authorization: Bearer <key>` to Artifactory to download the artifact.
- `no_log: true` ensures Jenkins console output never shows the key or response headers.

---

## 3. Repository layout

```text
ansible_learning/
├── ansible.cfg                 # Ansible defaults
├── .gitignore                  # Blocks .vault_pass and unencrypted vars
├── GUIDE.md                    # General Ansible Vault starter
├── GUIDE_JENKINS_ARTIFACTORY_VAULT.md  # This document
├── inventory/
│   └── hosts.yml               # Targets (localhost for artifact download)
├── group_vars/
│   └── all/
│       ├── vars.yml.example    # Template: copy, fill, encrypt
│       └── vault.yml           # Encrypted: committed to git
├── playbook.yml                # Playbook: download artifact from Artifactory
└── Jenkinsfile                 # Declarative pipeline
```

---

## 4. Phase 0 — Project setup (do once)

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

1. Log in to JFrog Artifactory (your company instance).
2. Go to **User Profile → Edit Profile → API Key**.
3. Copy the API key. Example format:

   ```
   AKCp8jwmV5Qi4m8...
   ```

4. Treat it like a password. Do not email it, do not paste it in Slack, do not commit it.

5. Note the artifact you want to download (example):

   ```
   URL: https://yourcompany.jfrog.io/artifactory/generic-local/myapp/myapp-1.0.0.zip
   ```

---

## 6. Phase 2 — Store the API key in Ansible Vault

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
artifactory_artifact_path: "myapp/myapp-1.0.0.zip"
artifactory_dest: "./downloads/myapp-1.0.0.zip"
```

### 6.2 Create the vault password file (local only)

```bash
printf '%s' "your-chosen-vault-password" > .vault_pass
chmod 600 .vault_pass
```

> `.vault_pass` is gitignored. Jenkins will NOT use this file directly; Jenkins will inject the vault password from a credential.

### 6.3 Encrypt the file

```bash
ansible-vault encrypt group_vars/all/vars.yml
```

The file on disk now starts with:

```text
$ANSIBLE_VAULT;1.1;AES256
```

### 6.4 Commit the encrypted file

```bash
git add group_vars/all/vars.yml
git commit -m "Add encrypted JFrog Artifactory vault variables"
git push
```

> Never commit `group_vars/all/vars.yml` (unencrypted). Never commit `.vault_pass`.

---

## 7. Phase 3 — Write the Ansible playbook

`playbook.yml`:

```yaml
---
- name: Download artifact from JFrog Artifactory using Vaulted API Key
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    artifactory_full_url: "{{ artifactory_url }}/{{ artifactory_repo }}/{{ artifactory_artifact_path }}"

  tasks:
    - name: Fail early if API key is missing
      ansible.builtin.fail:
        msg: "artifactory_api_key is not defined. Did you encrypt group_vars/all/vars.yml?"
      when: artifactory_api_key is undefined

    - name: Ensure download directory exists
      ansible.builtin.file:
        path: "{{ artifactory_dest | dirname }}"
        state: directory
        mode: '0755'

    - name: Download artifact from Artifactory
      ansible.builtin.uri:
        url: "{{ artifactory_full_url }}"
        method: GET
        return_content: true
        validate_certs: true
        headers:
          Authorization: "Bearer {{ artifactory_api_key }}"
        dest: "{{ artifactory_dest }}"
      register: artifactory_download
      failed_when: false
      no_log: true   # hides request headers and response from Jenkins logs

    - name: Report download status (no secrets printed)
      ansible.builtin.debug:
        msg: >
          Artifact download status: {{ artifactory_download.status | default('unknown') }}
          ({{ artifactory_download.msg | default('') }})
      when: artifactory_download.status == 200

    - name: Fail if download failed
      ansible.builtin.fail:
        msg: "Failed to download artifact from Artifactory. Status: {{ artifactory_download.status | default('unknown') }}"
      when: artifactory_download.status != 200
```

### Key security settings explained

| Setting | Purpose |
|---------|---------|
| `no_log: true` | Stops Ansible from printing the request/response (which would contain the API key) in Jenkins console output |
| `failed_when: false` on download | Lets us inspect the status and fail with a clean message instead of an ugly Ansible exception |
| `dest:` on `uri` module | Writes the artifact directly to disk instead of keeping it in memory |
| Never `debug: msg: "{{ artifactory_api_key }}"` | Would leak the secret to anyone reading Jenkins build logs |

### Run locally to test

```bash
ansible-playbook playbook.yml
```

Expected output (with a valid key):

```
TASK [Download artifact from Artifactory] *************************************
ok: [localhost]

TASK [Report download status] *************************************************
ok: [localhost] => {
    "msg": "Artifact download status: 200 (OK)"
}
```

The file `./downloads/myapp-1.0.0.zip` should now exist on your machine.

---

## 8. Phase 4 — Jenkins integration

Jenkins is the orchestrator. It checks out the repo, receives the vault password from a credential, and runs the playbook.

### 8.1 Install required Jenkins plugins

Manage Jenkins → Plugins → Available plugins → Install:

- **Ansible** (to run ansible-playbook)
- **Credentials Binding** (to inject secrets into the build)
- **Plain Credentials** (to store the vault password)

### 8.2 Create the Jenkins credential

Manage Jenkins → Manage Credentials → System → Global credentials → Add Credentials:

| Field | Value |
|-------|-------|
| Kind | Secret text |
| Scope | Global |
| Secret | `your-chosen-vault-password` (same as `.vault_pass`) |
| ID | `ansible-vault-password` |
| Description | Ansible Vault password for JFrog Artifactory |

### 8.3 Create the `Jenkinsfile`

```groovy
pipeline {
    agent any

    environment {
        // The vault password is injected as an environment variable by Jenkins
        VAULT_PASSWORD = credentials('ansible-vault-password')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Ansible') {
            steps {
                sh 'pip install ansible'
            }
        }

        stage('Write vault password file') {
            steps {
                // Jenkins writes the secret to .vault_pass at runtime
                sh 'printf "%s" "${VAULT_PASSWORD}" > .vault_pass'
                sh 'chmod 600 .vault_pass'
            }
        }

        stage('Download Artifact from Artifactory') {
            steps {
                sh 'ansible-playbook playbook.yml'
            }
        }
    }

    post {
        always {
            // Clean up the vault password file from the workspace
            sh 'rm -f .vault_pass'
        }
    }
}
```

### 8.4 Create the Jenkins job

1. **New Item** → Enter name (e.g. `download-artifactory-artifact`) → **Pipeline** → OK.
2. Under **Pipeline**, select **Definition: Pipeline script from SCM**.
3. SCM: **Git**.
4. Repository URL: your repo URL.
5. Branch: `*/main` (or your branch).
6. Script Path: `Jenkinsfile`.
7. Save.

### 8.5 Run the job

Click **Build Now**. Jenkins will:

1. Check out your repo (including the encrypted `vault.yml`).
2. Install Ansible.
3. Write `.vault_pass` from the Jenkins credential `ansible-vault-password`.
4. Run `ansible-playbook playbook.yml`.
5. Ansible decrypts the vault, reads `artifactory_api_key`, downloads the artifact.
6. Jenkins removes `.vault_pass` from the workspace after the job.

---

## 9. Phase 5 — Rotate the API key

API keys expire or get compromised. Here is how to rotate without touching Jenkins job configuration.

### Scenario A: New JFrog API key, same vault password

```bash
# 1. Get your new Artifactory API key from JFrog UI
# 2. Edit the encrypted vault file
ansible-vault edit group_vars/all/vars.yml

# 3. Replace the old artifactory_api_key with the new one
# 4. Save and exit the editor (file is re-encrypted automatically)
# 5. Commit and push
git add group_vars/all/vars.yml
git commit -m "Rotate JFrog Artifactory API key"
git push
```

Jenkins will pick up the new key on the next build. No Jenkins credential changes needed.

### Scenario B: Change the vault password itself

```bash
# 1. Rekey the vault file
ansible-vault rekey group_vars/all/vars.yml

# 2. Update local .vault_pass
printf '%s' "new-vault-password" > .vault_pass
chmod 600 .vault_pass

# 3. Update Jenkins credential
#    Manage Jenkins → Manage Credentials → ansible-vault-password → new-vault-password

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
| 6 | `ansible-vault-password` is a Jenkins Secret text credential, not in Jenkinsfile | ☐ |
| 7 | `post { always { rm -f .vault_pass } }` cleans up the password file | ☐ |
| 8 | You can run `ansible-vault view` and see your key | ☐ |

---

## 11. Troubleshooting

| Problem | Likely cause | Fix |
|---------|--------------|-----|
| `Vault password incorrect` | Wrong `.vault_pass` or wrong Jenkins credential ID | Verify the password matches what was used during `ansible-vault encrypt` |
| `artifactory_api_key is not defined` | Vault not loaded or file not encrypted properly | Run `ansible-vault view group_vars/all/vars.yml` to confirm contents |
| `401 Unauthorized` from Artifactory | API key expired or wrong repository path | Rotate the key in JFrog, update vault, re-run Jenkins job |
| `404 Not Found` | Wrong `artifactory_repo` or `artifactory_artifact_path` | Verify the artifact URL in a browser or curl |
| API key appears in Jenkins console output | Missing `no_log: true` or a `debug` task printing the key | Add `no_log: true` to all tasks using the key; remove debug tasks |

---

## 12. One-page cheat sheet

```bash
# Setup (local)
printf '%s' "mypassword" > .vault_pass && chmod 600 .vault_pass

# Vault lifecycle
ansible-vault encrypt group_vars/all/vars.yml
ansible-vault view group_vars/all/vars.yml
ansible-vault edit group_vars/all/vars.yml
ansible-vault rekey group_vars/all/vars.yml

# Run playbook locally
ansible-playbook playbook.yml

# Jenkins credential ID used in Jenkinsfile
# ansible-vault-password
```

---

## 13. Remember this forever

> **Ansible Vault = encrypted file + password.**  
> The encrypted file goes in git. The password goes in `.vault_pass` (local) or a Jenkins Secret text credential (CI).  
> Jenkins injects the password at runtime, writes `.vault_pass`, runs the playbook, then deletes `.vault_pass`.  
> The playbook reads the variable and sends `Authorization: Bearer <key>` to Artifactory.  
> `no_log: true` keeps the key out of Jenkins console logs.  
> Rotate by editing the vault, not the playbook or Jenkinsfile.

If you remember nothing else, remember: **secrets are encrypted at rest, never logged, injected at runtime, and rotated through the vault.**
