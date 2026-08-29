# OCI Always Free Arm (A1.Flex) VM Creator

[![Try to Create OCI VM](https://github.com/lialan213-eng/oci-free-arm-instance/actions/workflows/create-vm.yml/badge.svg)](https://github.com/lialan213-eng/oci-free-arm-instance/actions/workflows/create-vm.yml)

This repository contains two GitHub Actions workflows that automatically try to provision an "Always Free" `VM.Standard.A1.Flex` (Arm) compute instance in your Oracle Cloud Infrastructure (OCI) account, and report back to Discord once a day.

This is necessary because the "Always Free" Arm instances are a popular resource and are often unavailable due to high demand, resulting in an `"Out of host capacity."` error. You can try to upgrade to `pay as you go` plan which has a very good chance of getting available instance. Be sure to remain in free limits and check your costs frequently.

## What's in here

| Workflow | File | Schedule | Purpose |
| --- | --- | --- | --- |
| Try to Create OCI VM | `.github/workflows/create-vm.yml` | `7-57/10 * * * *` (every 10 min) | Retries the launch until it succeeds |
| Daily Summary - OCI VM Monitor | `.github/workflows/heartbeat.yml` | `0 6 * * *` (14:00 Beijing time) | Posts a 24-hour breakdown to Discord |

## Features

* **Fully Automated:** Runs entirely within GitHub Actions. You don't need to run anything on your local machine.
* **Persistent:** The workflow runs on a 10-minute schedule, offset from the top of the hour to reduce GitHub scheduler delays, continuously retrying until it successfully provisions your VM.
* **Single bundled secret:** All OCI settings live in one `OCI` repository secret instead of a dozen separate ones. Both `KEY=value` and `KEY: value` formats are accepted.
* **Validated before use:** OCID prefixes, region, availability domain, fingerprint and SSH key formats are checked before any API call is made, and every value is masked in the logs.
* **Idempotent:** If a `coolify-vm` instance already exists (anything not `TERMINATED`), no launch request is sent.
* **Self-disabling:** As soon as the VM exists, the workflow disables itself, so it can never create a second one.
* **Secure:** All sensitive credentials, keys, and IDs are stored in encrypted GitHub Secrets. The repository itself contains no private information and is safe to be public.
* **Fast:** Uses GitHub's caching to store the `oci-cli` installation, so subsequent runs are much faster.
* **Daily report:** A Discord summary classifies the last 24 hours into created / no capacity / quota exceeded / other OCI errors.

---

## How to Use

To use this, you need to **Fork** this repository and set up your OCI credentials as GitHub Secrets.

### Prerequisites

* An Oracle Cloud Infrastructure (OCI) "Always Free" account.
* A GitHub account.
* A Discord server/channel to receive the daily summary (optional).

---

## Step 1: Fork This Repository

Click the **"Fork"** button at the top-right of this page. This will create a copy of this repository in your own GitHub account. All the following steps will be done on **your fork**.

## Step 2: Gather Your OCI Information

You need **10 pieces of information** from your OCI account and your computer.

### A. Core IDs (Tenancy, User, Region)

1.  Log in to your OCI Console.
2.  **Tenancy OCID:** Click your **Profile icon** (top right) -> **Tenancy: [your\_tenancy\_name]**.
    * Copy the **OCID** value. This is your `OCI_CLI_TENANCY`.
    * *Note: For "Always Free" accounts, this is also your `OCI_COMPARTMENT_ID`.*
3.  **User OCID:** Click your **Profile icon** -> **User Settings**.
    * Copy the **OCID** value. This is your `OCI_CLI_USER`.
4.  **Region Identifier:** Look in the top-right corner of the console (e.g., "Singapore").
    * Hover over it or click it to find the identifier (e.g., `ap-singapore-1`). This is your `OCI_CLI_REGION`.

### B. OCI API Key (Private Key & Fingerprint)

1.  On the same **User Settings** page, click **"API Keys"** from the left-hand menu.
2.  Click the **"Add API Key"** button.
3.  Select **"Generate API Key Pair"**.
4.  Click **"Download Private Key"** and save the `oci_api_key.pem` file. **Do not lose this file.**
5.  Click the **"Add"** button.
6.  A "Configuration File Preview" will pop up. From this box, copy the `fingerprint` value. This is your `OCI_CLI_FINGERPRINT`.

### C. VM-Specific IDs (Subnet, Image, AD)

1.  **Subnet ID:**
    * Go to the OCI Console menu (☰) -> **Networking** -> **Virtual Cloud Networks**.
    * Click on your VCN (there is likely one default VCN).
    * Click on **"Subnets"** in the left menu.
    * Click on your public subnet (e.g., `Public Subnet ...`).
    * Copy the **OCID** of the subnet. This is your `OCI_SUBNET_ID`.
2.  **Availability Domain (AD) Name:**
    * Go to the OCI Console menu (☰) -> **Compute** -> **Instances**.
    * Click **"Create Instance"**.
    * In the **"Placement"** section, look at the **"Availability Domain"** dropdown. You likely only have one.
    * Copy its name *exactly* as it appears (e.g., `KClJ:AP-SINGAPORE-1-AD-1`). This is your `AD_NAME`.
3.  **Image ID:**
    * On the same "Create Instance" page, in the **"Image and shape"** section, click **"Change Image"**.
    * Select **"Canonical Ubuntu"** (or another OS of your choice).
    * Click the name of the image (e.g., "Canonical Ubuntu 22.04"). A details panel will slide out.
    * Copy the **OCID** of the image. This is your `IMAGE_ID`.
    * You can now cancel the "Create Instance" wizard.
    * If you still can not find the image, check this website and choose your image and copy ocid mentioned according to region. <https://docs.oracle.com/en-us/iaas/images/>
4.  **OCPUs and RAM**
    * The default settings in this action is to provision instance with 2 OCPUs and 12 GB memory.
    * Change the `SHAPE_CONFIG` env in the **Launch VM** step of `.github/workflows/create-vm.yml`:
      `--shape-config '{"ocpus":2,"memoryInGBs":12}'`
5.  **Boot Volume and Name**
    * The default settings in this action is to provision instance with `100` GB boot volume with name `coolify-vm`.
    * Change the boot volume with `--boot-volume-size-in-gbs 100`.
    * Change the instance name with `--display-name "coolify-vm"`.
    * If you rename it, also update the `display-name` filter in the **Check for an existing instance** step.

### D. Your SSH Public Key

This is the key you will use to log in to your new server.

1.  Open a terminal on your computer.
2.  Check if you have a key: `cat ~/.ssh/id_rsa.pub`
3.  **If it shows a key:** Copy the entire output (it starts with `ssh-rsa...`). This is your `SSH_PUBLIC_KEY`.
4.  **If it shows "No such file":** Run `ssh-keygen -t rsa -b 2048`. Press Enter three times to accept the defaults. Then, run `cat ~/.ssh/id_rsa.pub` again and copy the key.
5.  Only `ssh-rsa` and `ssh-ed25519` keys are accepted.

---

## Step 3: Create the `OCI` Secret

All 10 values go into **one** repository secret named `OCI`. Paste a block like this into the secret value:

```ini
OCI_CLI_USER=ocid1.user.oc1..aaaaaaaa...
OCI_CLI_FINGERPRINT=aa:bb:cc:11:22:33:44:55:66:77:88:99:aa:bb:cc:dd
OCI_CLI_TENANCY=ocid1.tenancy.oc1..aaaaaaaa...
OCI_CLI_REGION=ap-singapore-1
OCI_COMPARTMENT_ID=ocid1.tenancy.oc1..aaaaaaaa...
OCI_SUBNET_ID=ocid1.subnet.oc1.ap-singapore-1.aaaaaaaa...
AD_NAME=KClJ:AP-SINGAPORE-1-AD-1
IMAGE_ID=ocid1.image.oc1.ap-singapore-1.aaaaaaaa...
SSH_PUBLIC_KEY=ssh-rsa AAAAB3NzaC1yc2E... you@host
OCI_CLI_KEY_CONTENT=|
  -----BEGIN PRIVATE KEY-----
  MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQ...
  ...
  -----END PRIVATE KEY-----
```

Notes on the format:

* `KEY=value` and `KEY: value` are both accepted, as is an `export ` prefix.
* Values wrapped in single or double quotes are unwrapped.
* The private key can be a YAML `|` block (as above), a single line using literal `\n`, or base64 — all three are handled.
* **Alternative:** keep the private key in a separate secret named `OCI_CLI_KEY_CONTENT`, which overrides whatever is in the bundle. The bundle must still contain the `OCI_CLI_KEY_CONTENT` key (any non-empty placeholder works).

Every value is masked in the run logs, and the fields are validated against these rules before any request is sent:

| Field | Must look like |
| --- | --- |
| `OCI_CLI_USER` | `ocid1.user.` |
| `OCI_CLI_TENANCY` | `ocid1.tenancy.` |
| `OCI_COMPARTMENT_ID` | `ocid1.tenancy.` or `ocid1.compartment.` |
| `OCI_SUBNET_ID` | `ocid1.subnet.` |
| `IMAGE_ID` | `ocid1.image.` |
| `OCI_CLI_REGION` | e.g. `ap-singapore-1` |
| `AD_NAME` | e.g. `KClJ:AP-SINGAPORE-1-AD-1` |
| `OCI_CLI_FINGERPRINT` | 16 colon-separated hex pairs |
| `SSH_PUBLIC_KEY` | starts with `ssh-rsa` or `ssh-ed25519` |

## Step 4: Create a Discord Webhook (optional)

1.  Open your Discord server. Right-click on a channel name and click **"Edit Channel"**.
2.  Go to the **"Integrations"** tab.
3.  Click **"Webhooks"** -> **"New Webhook"**.
4.  Give it a name (e.g., "OCI Notifier") and click **"Copy Webhook URL"**.
5.  Add it as a repository secret named `DISCORD_WEBHOOK_URL`.

## Step 5: Configure GitHub Secrets

Go to your forked repository on GitHub: **Settings** -> **Secrets and variables** -> **Actions** -> **New repository secret**.

| Secret | Required | Value |
| --- | --- | --- |
| `OCI` | yes | The whole block from Step 3 |
| `OCI_CLI_KEY_CONTENT` | no | Only if you kept the private key out of the bundle |
| `DISCORD_WEBHOOK_URL` | no | The URL from Step 4 |

---

## Step 6: Run the Workflow

1.  Go to the **"Actions"** tab of your forked repository.
2.  In the left sidebar, click on **"Try to Create OCI VM"**.
3.  You will see a message: "This workflow has a `workflow_dispatch` event." Click the **"Run workflow"** button on the right, and then **"Run workflow"** again.

This will start the first run. From now on, the `schedule` will automatically run it every 10 minutes.

---

## Understanding the Results

Each run prints a single line that tells you exactly what happened:

```
OCI launch response: result=capacity code=LimitExceeded status=429 request_id=... message=...
```

| `result` | Meaning | What to do |
| --- | --- | --- |
| `created` | OCI accepted the launch | Nothing — the workflow disables itself |
| `existing` | A `coolify-vm` already exists | Nothing — the workflow disables itself |
| `capacity` | `Out of host capacity` | Wait; it retries every 10 minutes |
| — (red run) | Anything else, e.g. `LimitExceeded`, bad subnet, wrong image | Fix the config; the run fails loudly on purpose |

Once the VM exists the workflow **disables itself** (`Disable creation workflow after success` step), so it will never create a second instance. If you delete the VM and want to try again, re-enable it from the three-dot menu in the **Actions** tab.

Your VM will be provisioning in the OCI console. You can now log in using the SSH key you provided.

## Daily Discord Summary

`heartbeat.yml` runs at 14:00 Beijing time. It pulls the last 24 hours of `create-vm.yml` runs, downloads each run's log archive, and counts them by category:

* Workflow state, number of runs, and how many actually reached OCI
* 创建成功 / 主机容量不足 / A1 配额不足 / 其他 OCI 错误 / 已存在实例 / 未到创建步骤 / 日志不可用
* The result of the most recent run

It uses the built-in `GITHUB_TOKEN` for read access, so no extra token is needed. If `DISCORD_WEBHOOK_URL` is not set, the step skips quietly.

## License

This repository is available under the [MIT License](LICENSE).
