# GitHub Deploy User Setup Guide

*Single machine-user account for the Ubuntu server*

## Why a dedicated GitHub deploy user?

Your dad's guide describes **per-repo deploy keys** — one SSH keypair per repository, each scoped to exactly one repo. That approach is the most granular and is ideal for shared servers with multiple developers and many repos owned by different GitHub accounts.

For a simpler setup — where one person manages a small number of repos — a **dedicated GitHub machine user** offers the same security separation from your personal account, with less operational overhead.

### The problem with your personal SSH key on root

This is the same as in the deploy keys guide:

- Root's personal SSH key grants access to **every** repo your account can reach.
- If the server is compromised, all repos are exposed.
- If your personal account changes credentials, all deployments break.
- Code pulled as root executes hooks with full root privileges.

### How a deploy user solves this

A deploy user is a **separate GitHub account** (e.g. `hofnet-deploy`) that exists only for automated deployments. You add it as a **read-only collaborator** on the repos the server needs to pull.

| Benefit | Explanation |
|---|---|
| **Separated from personal account** | Your personal GitHub account is never tied to the server. If it changes, deployments are unaffected. |
| **Scoped access** | The deploy user only has access to repos where it is explicitly added as a collaborator. |
| **Single SSH key** | One keypair on the server works for all repos the deploy user can access — no per-repo aliases needed. |
| **Easy to revoke** | Remove the deploy user as collaborator from a repo, or delete the account entirely. |
| **No SSH config aliases** | Git remotes use standard `git@github.com:owner/repo.git` URLs. No host alias mapping required. |

### When to use deploy keys instead

Use per-repo deploy keys (your dad's guide) when:

- Multiple developers share a server and own repos under **different** GitHub accounts.
- You need per-repo key isolation (compromise of one key must not expose other repos).
- You have a large number of repos and want independent key rotation.

Use a single deploy user when:

- One person or team manages the server.
- You have a small number of repos (< 10).
- You want simplicity over maximum granularity.

---

## Environment

| Item | Details |
|---|---|
| **Server** | Ubuntu (hofnet-server) |
| **Deploy GitHub account** | A dedicated deployment account (`hofnet-deploy-bot`) |
| **Server user running deploys** | `root` (via deploy script) or service users |
| **Key storage** | `/root/.sshs/` | (files: `hofnet_machine_key` and `hofnet_machine_key.pub`)
| **Repos to deploy** | `Wisdom-Chicken/adagium` and `Bringolo/Artbots` (more wiil be added) |

---

## Step 1 — Create the GitHub deploy user

1. Go to [github.com/signup](https://github.com/signup).
2. Create a new account with a clear machine-user name, e.g. `hofnet-deploy-bot`.
3. Use a dedicated email address (e.g. `deploy@hofnet.nl` or a `+` alias like `laurens+deploy@gmail.com`).
4. **Do not** enable 2FA with a hardware key you might lose — use an authenticator app and store the recovery codes securely.

> **GitHub policy**: Machine users are allowed under GitHub's Terms of Service. They consume one seat in paid orgs but are free for personal/public repos.

---

## Step 2 — Generate the SSH keypair on the server

Generate a single Ed25519 keypair for the deploy user.

```bash
# Create the directory
sudo mkdir -p /etc/deploy-keys

# Generate the keypair
sudo ssh-keygen -t ed25519 -C "hofnet-deploy-bot" -f /root/.ssh/hofnet_machine_key -N ""

# Lock down permissions
sudo chmod 600 /root/.ssh/*
sudo chmod 700 /root/.ssh
```

The `-N ""` flag creates the key without a passphrase, which is required for automated deployments.

---

## Step 3 — Add the public key to the deploy GitHub account

Print the public key:

```bash
sudo cat /root/.ssh/hofnet_machine_key.pub
```

Then add it to the **deploy user's GitHub account** (not a specific repo):

1. Log in as `hofnet-deploy-bot` on GitHub.
2. Go to **Settings → SSH and GPG keys → New SSH key**.
3. Paste the public key. Give it a title like `hofnet-server`.

This single key now authenticates as the deploy user for **all repos** it has access to.

---

## Step 4 — Add the deploy user as collaborator

For each repo the server needs to pull:

1. Log in as **your personal account** (the repo owner).
2. Go to the repo → **Settings → Collaborators → Add people**.
3. Invite `hofnet-deploy-bot`.
4. Accept the invitation from the deploy user account.

> **Tip**: For read-only access, grant the **Read** role. The deploy user does not need write access.

### Current repos

| GitHub repo | Role |
|---|---|
| `Wisdom-Chicken/adagium` | Read |

Add more rows as you deploy more apps.

---

## Step 5 — Configure SSH on the server

Edit `/root/.ssh/config` (or the SSH config for whichever user runs deploys):

```
Host github.com
    HostName github.com
    User git
    IdentityFile /etc/deploy-keys/hofnet-deploy
    IdentitiesOnly yes
```

This tells SSH to **always** use the deploy key when connecting to `github.com`. No per-repo aliases needed.

> **Note**: If you also need your personal GitHub SSH access on this server (e.g. for interactive git work), use a host alias for one of them:
>
> ```
> # Deploy (default for github.com)
> Host github.com
>     HostName github.com
>     User git
>     IdentityFile /root/.ssh/hofnet_machine_key
>     IdentitiesOnly yes
>
> ```

---

## Step 6 — Update existing git remotes (if needed)

If your repos already use standard `git@github.com:` URLs, **no changes are needed**. The SSH config handles key selection automatically.

Verify the current remote:

```bash
cd /srv/adagium && git remote -v
```

If it shows an HTTPS URL, switch to SSH:

```bash
cd /srv/adagium && git remote set-url origin git@github.com:Wisdom-Chicken/adagium.git
```

---

## Step 7 — Verify

Test that the deploy key authenticates correctly:

```bash
sudo ssh -T -i /etc/deploy-keys/hofnet-deploy-bot git@github.com
# Expected: Hi hofnet-deploy-bot! You've successfully authenticated, but GitHub does not provide shell access.
```

Then test a pull:

```bash
cd /srv/adagium && sudo git fetch origin
```

If both succeed, the deploy user is working.

---

## Deploy script adaptation

Your existing `deploy.sh` does `git fetch origin main` and `git pull origin main --ff-only` as root. Since root's SSH config now points to the deploy key, **no changes to the deploy script are needed**.

The key selection happens transparently via the SSH config.

---

## Adding a new app

When you deploy a new repo on the server:

1. Add `hofnet-deploy-net` as a **Read** collaborator on the new GitHub repo.
2. Clone or set the remote to `git@github.com:Owner/repo.git`.
3. Done. The same SSH key works automatically.

No new keypairs, no new SSH aliases.

---

## Cleanup

Once the deploy user is verified and working:

1. Remove your personal SSH key from the server (if it was added to root's `authorized_keys` or GitHub).
2. Ensure your personal GitHub account has no server-side SSH keys attached.
3. The server now authenticates to GitHub exclusively through the deploy user.

---

## Comparison: Deploy keys vs. Deploy user

| Aspect | Per-repo deploy keys | Single deploy user |
|---|---|---|
| **Keys to manage** | One per repo | One total |
| **SSH config entries** | One alias per repo | One entry |
| **Access scope** | Single repo per key | All repos where collaborator |
| **Blast radius if key leaked** | One repo | All collaborator repos |
| **Adding a new repo** | Generate key + add alias + add to repo | Add as collaborator |
| **Removing a repo** | Delete key + alias | Remove collaborator |
| **Best for** | Multi-developer, many repos | Single developer, few repos |
