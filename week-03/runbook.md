# Week 3 Runbook: Deploying PropertyLite on EC2

**Goal:** Launch an EC2 instance that bootstraps and runs PropertyLite (Flask API) on port 8080 using a user data script, then practice EBS snapshots.

**Time:** ~20 minutes
**Cost:** Free Tier (`t2.micro` / `t3.micro`). Terminate everything when finished.

---

## Prerequisites

- AWS account with console access
- Terminal with `ssh` and `curl`
- `app.py` from Week 0 (needed for the user data script)
- Your current public IP (run `curl ifconfig.me`)

## Reference values

| Item | Value |
|---|---|
| Instance name | `property-api-01` |
| AMI | Amazon Linux 2023 |
| Instance type | `t2.micro` or `t3.micro` |
| Key pair | `training-key` (RSA, `.pem`) |
| SSH user | `ec2-user` |
| App port | `8080` |
| App directory | `/opt/property-api` |
| App log | `/var/log/property-api.log` |

---

## Lab 3.1: Deploy PropertyLite with a bootstrap script

### 1. Launch the instance

1. Console → **EC2** → **Launch instance**.
2. **Name:** `property-api-01`.
3. **AMI:** Amazon Linux 2023 (Free Tier eligible).
4. **Instance type:** `t2.micro` or `t3.micro`.
5. **Key pair:** Create new key pair → name `training-key` → type **RSA** → format **.pem** → download.
   - Move it somewhere permanent, e.g. `~/.ssh/training-key.pem`.
   - **There is no way to re-download this file.** If you lose it, you must create a new key pair.

### 2. Configure the security group

Under **Network settings** → **Create security group**, add two inbound rules:

| Type | Port | Source | Why |
|---|---|---|---|
| SSH | 22 | **My IP** | Admin access only from your machine |
| Custom TCP | 8080 | Anywhere (`0.0.0.0/0`) | PropertyLite's port (not the default HTTP port 80) |

> Do not set SSH to "Anywhere". It leaves your server open to the whole internet.

### 3. Add the bootstrap script

Expand **Advanced details** → scroll to **User data** and paste the script below. It runs once as root on first boot and makes the instance self-contained (app code and sample data are inlined).

```bash
#!/bin/bash
yum update -y
yum install -y python3 python3-pip
pip3 install flask==3.0.3

mkdir -p /opt/property-api

cat > /opt/property-api/app.py << 'PYEOF'
# ... paste the full app.py from Week 0 here ...
PYEOF

cat > /opt/property-api/rets_property_sample.csv << 'CSVEOF'
L_ListingID,L_City,L_Keyword2,LM_Dec_3,LM_Int2_3,L_SystemPrice,L_Status
R100234,Sacramento,3,2.0,1450,450000,Active
R100235,Fresno,4,2.5,1900,395000,Active
R100236,Sacramento,2,1.0,900,299000,Pending
CSVEOF

cd /opt/property-api && nohup python3 app.py > /var/log/property-api.log 2>&1 &
```

**What each part does**

| Section | Purpose |
|---|---|
| `yum update -y` | Patches the OS |
| `yum install ... python3 python3-pip` | Installs Python and pip |
| `pip3 install flask==3.0.3` | Installs a pinned Flask version for reproducibility |
| `cat > ... << 'PYEOF'` | Writes `app.py` to disk (quoted heredoc prevents shell expansion) |
| `nohup python3 app.py ... &` | Starts the app in the background, logging to `/var/log/property-api.log` |

### 4. Launch and connect

1. Click **Launch instance**, wait for **Instance state: Running** and status checks to pass, then copy the **public IPv4 address**.
2. Lock down the key file (SSH refuses keys that are too permissive):
   ```bash
   chmod 400 ~/.ssh/training-key.pem
   ```
3. Connect:
   ```bash
   ssh -i ~/.ssh/training-key.pem ec2-user@<public-ip>
   ```

### 5. Verify

From your **laptop** (not the instance):

```bash
curl http://<public-ip>:8080/health
# expected: {"status":"ok"}

curl http://<public-ip>:8080/properties/R100234
# expected: the Sacramento listing
```

**Success criteria:** both calls return the expected output. This is the same app from Week 0, now on real infrastructure.

> Bootstrap can take 1-2 minutes after the instance shows "Running". If `curl` fails right away, wait and retry.

**Screenshot:** _(insert screenshot of both curl responses here)_

---

## Lab 3.2: EBS snapshot practice

1. Console → **EC2** → **Volumes** → select the **root volume** of `property-api-01`.
2. **Actions** → **Create snapshot**.
3. Go to **Snapshots** and wait for status **completed**.

**Success criteria:** the snapshot appears in **EC2 → Snapshots** with status `completed`.

**What happens if you create a new volume from it?** You get an exact copy of the original volume as it was when the snapshot was taken. You can attach it to an instance to restore or clone that disk.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| SSH: `UNPROTECTED PRIVATE KEY FILE` / "permissions too open" | Key file permissions | `chmod 400 ~/.ssh/training-key.pem` |
| SSH hangs, then times out (worked yesterday) | Your public IP changed; the "My IP" rule is a snapshot, not a live lookup | Compare `curl ifconfig.me` to the SG inbound rule, then edit the SSH rule's source to your current IP |
| SSH hangs on a brand-new instance | Instance not running, or no internet gateway/route in the subnet | Check instance state; confirm the default VPC's route table has a route to an internet gateway |
| `curl` to port 8080 hangs | Port 8080 not open, or port 80 opened by mistake | Add Custom TCP 8080 inbound rule to the security group |
| `curl` connection refused | App not running yet, or crashed | SSH in and check `cat /var/log/property-api.log`; confirm `app.py` pasted correctly |
| Lost the `.pem` file | Can't be re-downloaded | Create a new key pair; for existing instances, launch a new one |

**Useful checks on the instance**

```bash
ps aux | grep app.py                         # is the app running?
sudo cat /var/log/cloud-init-output.log      # did user data finish/error?
cat /var/log/property-api.log                # app output
```

---

## Cleanup

1. EC2 → **Instances** → select `property-api-01` → **Instance state** → **Terminate**.
2. Check the **Volumes** tab to confirm the root volume was deleted.
3. Delete the snapshot from Lab 3.2 if you no longer need it (snapshots incur storage charges).
4. Keep `training-key.pem`. Later labs reuse it.

---

## Key takeaways

- **User data** automates setup at first boot, so instances can be rebuilt identically.
- **Security groups** are stateful and instance-level. Scope SSH to your IP and expose only the ports the app needs.
- **"My IP" is static**: if your network changes, update the rule.
- **Snapshots** back up EBS volumes and can be restored as new volumes.
- Always terminate resources you aren't using.