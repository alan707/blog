---
title: "How to Get a Free Computer from Oracle (After 50 People Beat You to It)"
date: 2026-05-03
tags: ["oracle-cloud", "free-tier", "arm", "automation", "infrastructure"]
draft: true
---

### TL;DR

Oracle gives you a free 4-core/24GB ARM server, but everyone wants one and the "Create" button fails with "Out of host capacity" roughly 50 times in a row before it works. This post shows you how to run a script that hunts for capacity on your behalf — 24/7 — from a *second* free Oracle VM. Copy-paste, walk away, come back to a server.

### The free tier nobody tells you about

Oracle Cloud's Always Free tier is the only major cloud offering a real, persistent server — not a sandbox, not a time-limited trial, not a shared-CPU micro.

| Cloud | Always Free compute | RAM | Notes |
|---|---|---|---|
| Oracle Cloud (OCI) | 4 OCPU ARM (A1.Flex) + 2 AMD x86 micros | 24 GB (ARM) + 2×1 GB | Persistent, full Linux, no expiry |
| AWS | t2.micro or t3.micro | 1 GB | 12-month trial only |
| GCP | e2-micro (shared) | ~1 GB | One instance, US regions only |
| DigitalOcean | None | — | No always-free compute |
| Azure | B1s (12-month trial) | 1 GB | Trial only |

The ARM instance — `VM.Standard.A1.Flex` — is 4 OCPU Ampere and 24 GB RAM. That's a serious machine. The catch is that Oracle knows it's a serious machine, so does every other person who's read this, and the capacity is chronically oversold in popular regions.

### ⚠️ Pick your region before you sign up

**This is the single most consequential decision in the post. You cannot change your home region on the free tier without upgrading to a paid account.**

As of May 2026, regions that have historically had more A1.Flex headroom include Phoenix (`us-phoenix-1`), Frankfurt (`eu-frankfurt-1`), and London (`uk-london-1`). Ashburn (`us-ashburn-1`) and San Jose (`us-sanjose-1`) tend to be the most crowded. Capacity ebbs and flows — don't treat any specific region recommendation as permanent truth — but if you're in the US and Phoenix is an option, start there.

### Step 1: Sign up for OCI Free Tier

Go to [cloud.oracle.com](https://cloud.oracle.com) and create an account. A few things to know upfront:

- **Credit card required** for identity verification. Oracle will authorize a small temporary charge (typically $1) but won't bill you for Always Free resources.
- **SMS verification** can take anywhere from a few minutes to overnight depending on your carrier. Don't panic if you're waiting.
- **Pick your home region on the last step of sign-up.** This is the permanent choice described above.

### Step 2: Grab your Tenancy OCID and User OCID

OCIDs (Oracle Cloud Identifiers) look like this: `ocid1.tenancy.oc1..aaaaaaaa…`. You'll need two of them.

**Tenancy OCID:**
1. Click the profile icon in the top-right corner
2. Click "Tenancy: \<your-tenancy-name\>"
3. Copy the OCID shown at the top of the page

**User OCID:**
1. Click the profile icon again
2. Click "My profile"
3. Copy the OCID shown at the top of the page

Keep both handy — you'll paste them into the CLI setup in the next step.

### Step 3: Install OCI CLI and configure auth

**Install:**

```bash
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"
```

After it finishes, add the CLI to your PATH (the installer will tell you the exact path, but this is the typical default):

```bash
export PATH="$HOME/bin:$PATH"
```

Add that line to your `~/.bashrc` or `~/.zshrc` so it persists across sessions.

**Configure:**

```bash
oci setup config
```

This walks you through an interactive setup. When it asks:
- **User OCID** → paste your User OCID from Step 2
- **Tenancy OCID** → paste your Tenancy OCID from Step 2
- **Region** → enter your home region (e.g. `us-phoenix-1`)
- **Generate a new API signing key pair?** → Y
- **Passphrase for the private key** → leave blank (press Enter)

After the wizard finishes, it will show you your public key. Copy it:

```bash
cat ~/.oci/oci_api_key_public.pem
```

Now upload it to the console:
1. Profile icon → "My profile"
2. "Tokens and keys" in the left sidebar
3. "Add API key"
4. Paste the public key content
5. Click "Add"

**Verify it works:**

```bash
oci iam region list
```

You should see a JSON list of OCI regions. If you get an auth error, double-check that the public key was uploaded correctly and that the OCIDs match.

### Step 4: Bootstrap the x86 hunter VM

Here's the core trick: instead of running the retry loop from your laptop (where closing the lid kills it) or Cloud Shell (which disconnects after ~20 minutes of idle and token-expires after an hour), we spin up a second Always Free VM — an `VM.Standard.E2.1.Micro` (1 OCPU AMD x86, 1 GB RAM) — whose sole job is to run the retry loop 24/7.

The micro VM has plentiful capacity because nobody's fighting over 1-core x86 boxes. It's perpetually free. And it stays awake while you don't.

Download and run the bootstrap script:

```bash
curl -O https://gist.github.com/alan707/90eb72c691b4f58a89b164af97461422/raw/bootstrap-hunter.sh
chmod +x bootstrap-hunter.sh
SSH_KEY_FILE=/path/to/your/public_key.pub ./bootstrap-hunter.sh
```

The script discovers your tenancy, availability domain, and latest Ubuntu 24.04 image automatically from your OCI CLI config. It creates a VCN, internet gateway, subnet, and launches the instance. When it finishes, it prints:

```
===== ready =====
public IP: <your-public-ip>
ssh:       ssh ubuntu@<your-public-ip>
```

When you're done with the hunter VM (after the ARM VM is running), you can tear down all the resources it created with:

```bash
./bootstrap-hunter.sh --teardown
```

### Step 5: Create the ARM stack in Resource Manager

We use Oracle's Resource Manager (managed Terraform) rather than a bare `oci compute instance launch` loop because Resource Manager Apply jobs have a clean `SUCCEEDED`/`FAILED` state that's trivial to poll. When a raw instance launch fails with capacity issues, the error handling is messier.

**Get the Terraform file:**

```bash
curl -O https://gist.github.com/alan707/0c038fc37ecfe8555842b66764d6148b/raw/main.tf
zip arm-stack.zip main.tf
```

**Create the stack:**

1. OCI Console → Developer Services → Resource Manager → Stacks → "Create Stack"
2. Choose "My configuration" and upload `arm-stack.zip`
3. Click through to the Variables page and fill in:
   - **tenancy\_ocid** — your Tenancy OCID from Step 2
   - **ssh\_public\_key** — paste the contents of your SSH public key (`cat ~/.ssh/your_key.pub`)
   - **region** — your home region (e.g. `us-phoenix-1`)
   - **availability\_domain** — your AD name, e.g. `abcd:PHX-AD-1` (visible in the console under Identity → Availability Domains)
   - **image\_ocid** — the ARM-compatible image OCID for your region. In the console: Compute → Images → filter by "Canonical Ubuntu" + "24.04" + shape "VM.Standard.A1.Flex", copy the OCID.
   - **ocpus / memory\_in\_gbs** — defaults are 2 OCPU / 16 GB. The Always Free allotment is 4 OCPU / 24 GB total across all A1 instances. Adjust as you like.
4. On the review page, click "Create" — **do not** click "Run Apply". The retry script does that.
5. Copy the Stack OCID from the stack detail page (it looks like `ocid1.ormstack.oc1.phx.…`).

### Step 6: Run the retry loop on the hunter

SSH into your hunter VM:

```bash
ssh ubuntu@<hunter-public-ip>
```

Install OCI CLI on it (same command as Step 3), configure it the same way (`oci setup config`), and upload the same API public key.

One practical shortcut: if you configured API-key auth on your laptop (not browser-session auth), you can copy your OCI config directory directly:

```bash
scp -r ~/.oci ubuntu@<hunter-public-ip>:~/.oci
```

Then download and run the retry loop:

```bash
curl -O https://gist.github.com/alan707/87ce1959f871861a54cfe7c04358a3cb/raw/run_arm.sh
chmod +x run_arm.sh
```

Run it under tmux so it survives SSH disconnects:

```bash
tmux new -s arm
STACK_OCID=ocid1.ormstack.oc1.<your-stack-ocid> ./run_arm.sh
```

Press `Ctrl-b d` to detach from tmux and close your SSH session. The loop keeps running. To check back:

```bash
ssh ubuntu@<hunter-public-ip>
tmux attach -t arm
```

### What to expect while you wait

Capacity comes in waves. It could be a few hours; it could be a few days. The script handles the two failure modes gracefully:

- **429 / rate-limited** — Oracle's Resource Manager API throttled you. The script sleeps 10 minutes and retries.
- **"Out of host capacity"** — no A1.Flex hardware available right now. The script sleeps 6 minutes and tries again.

You'll see attempt counters tick up in the tmux window. Ignore them. When capacity lands, the script prints the new instance's public IP and exits cleanly.

### Once you have it: now what?

A 4-core/24GB ARM box for $0/month is genuinely useful:

- **[Tailscale](https://tailscale.com)** or **[Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/)** for stable access without exposing a public IP
- **Docker host** for running side projects without paying DigitalOcean
- **Personal media server** — Plex, Jellyfin, or Navidrome on a box that doesn't spin down
- **SearxNG or other self-hosted tools** — 24 GB of RAM goes a long way for things like local LLM inference

### One closing thought

The actual trick here isn't Oracle-specific. It's: find an API that gives you a clean success/failure signal, poll it from a persistent machine, sleep between attempts. This pattern works anywhere resources are capacity-constrained — spot instances, GPU reservations, whatever. You're not just getting a free server. You're learning a small but real ops move.
