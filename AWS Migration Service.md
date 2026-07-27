# AWS Migration Service (MGN) Lab — Simple Step-by-Step

Just follow these steps in order. Don't jump ahead.

---

## WHAT WE'RE DOING (in plain words)

1. We create one EC2 server → pretend it's an "old server" (source).
2. We install a small agent on it → it starts copying its data to AWS.
3. We wait till copying is 100% done.
4. We launch a **test copy** of the server → check it works.
5. We do the **real switch (cutover)** → this creates the final new server.
6. We clean everything up so we don't get billed.

---

## STEP 1: Create the "old server" (Source EC2)

1. Go to **EC2 → Launch Instance**.
2. Name it: `source-server-01`
3. Pick **Amazon Linux 2023**.
4. Instance type: `t3.medium`
5. Create/select a **key pair** → download it.
6. Leave network as default VPC (or pick your own).
7. Security group → Allow:
   - SSH (port 22) — from My IP
   - HTTP (port 80) — from My IP
8. Click **Launch Instance**.
9. Wait 2–3 minutes → status shows **2/2 checks passed**.



---

## STEP 2: Put a test webpage on it

Connect to the instance (EC2 → Connect), then run:

```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
echo "<h1>Hello from OLD server</h1>" | sudo tee /var/www/html/index.html
```

Copy the instance's **Public IP**, open it in browser → you should see "Hello from OLD server". This proves the server works before we migrate it.


<img width="2885" height="1137" alt="image" src="https://github.com/user-attachments/assets/30244044-41d0-4a7c-8f8b-50da8bbb4673" />

---

## STEP 3: Turn on MGN in your target region

1. Choose which region the **new/migrated server** should live in (can be same region, that's fine for a lab).
2. Go to **AWS Application Migration Service** (search "MGN" in console search bar).
3. Click **Get started**.
4. It will ask for a **Replication Settings Template**:
   - Staging subnet → pick any subnet in your VPC
   - Instance type → leave default
   - Volume type → gp3
   - Encryption → leave default (enabled)
5. Click **Save / Create**.

This is a one-time setup per region.

---

## STEP 4: Give the source server permission to talk to MGN

1. Go to **IAM → Roles → Create role**.
2. Trusted entity: **AWS service → EC2**.
3. Attach policy: `AWSApplicationMigrationAgentPolicy`
4. Name it: `mgn-agent-role` → Create role.
5. Go back to **EC2 → source-server-01 → Actions → Security → Modify IAM role**.
6. Attach `mgn-agent-role` → Save.

---

## STEP 5: Install the MGN Agent on the source server

1. In the MGN console → click **Add source server** (or "Get started" screen).
2. Choose **Linux agent-based**.
3. It shows you a command like this — copy it:

```bash
sudo su
wget -O ./aws-replication-installer-init.py https://aws-application-migration-service-<REGION>.s3.<REGION>.amazonaws.com/latest/linux/aws-replication-installer-init.py
python3 aws-replication-installer-init.py --region <REGION> --no-prompt
```

4. Paste and run it on `source-server-01` (SSH session from Step 2).
5. Wait till it says "Installation completed."

---

## STEP 6: Watch it start copying data

1. Go back to MGN console → **Source servers**.
2. Within a few minutes, `source-server-01` will show up.
3. Look at the **Data replication progress** column → it climbs from 0% to 100%.
4. **Wait here until it says "Ready for testing"** — don't move to the next step before this.

---

## STEP 7: Set up how the new server should launch

1. Click on `source-server-01` in the list.
2. Go to **Launch settings** tab → Edit.
3. Set:
   - Instance type → leave auto/right-sized, or pick `t3.medium`
   - Subnet → pick where the new server should launch
   - Security group → allow SSH + HTTP (same as Step 1)
4. Save.

---

## STEP 8: Launch a TEST server (practice run, safe, doesn't affect anything)

1. Select `source-server-01` (checkbox).
2. Click **Test and cutover → Launch test instances**.
3. Wait a few minutes → a new EC2 instance appears in your account.
4. Open its **Public IP** in browser → you should see "Hello from OLD server" — meaning the copy worked.
5. If it works: go back to MGN console → **Test and cutover → Mark as "Ready for cutover"**.
   (This automatically deletes the test server for you — no manual cleanup needed here.)

---

## STEP 9: Do the REAL cutover (this creates your final migrated server)

1. Select `source-server-01` again.
2. Click **Test and cutover → Launch cutover instances**.
3. Wait for the new instance to appear.
4. Open its Public IP → confirm "Hello from OLD server" shows up.
5. This is now your **real migrated server** — in a real migration, you'd point your domain/DNS to this new server now.

---

## STEP 10: Finish (finalize) the migration

1. Once you're happy everything works on the new server:
2. Go to **Test and cutover → Finalize cutover**.
3. Confirm.
4. This stops the copying process for good (you can't undo this — only do it once you're sure).

---

## STEP 11: Clean up (IMPORTANT — do this so AWS doesn't keep charging you)

Delete things in this order:

1. **Terminate** the new cutover EC2 instance (the migrated one) — EC2 → Instances → Terminate.
2. **Terminate** `source-server-01` (the old/original one) — EC2 → Instances → Terminate.
3. In MGN console → select the source server → **Actions → Disconnect from service**.
4. Go to **EC2 → Volumes** and **EC2 → Snapshots** → delete any leftover ones from this lab.
5. Delete the IAM role `mgn-agent-role` (IAM → Roles).
6. Delete the security groups you made just for this lab.

---

## Quick recap (just the actions, in order)

```
1. Launch source EC2  →  2. Install web app on it
3. Turn on MGN + replication template
4. Create IAM role → attach to source EC2
5. Install replication agent on source EC2
6. Wait for 100% sync ("Ready for testing")
7. Set launch settings (subnet + SG)
8. Launch TEST instance → verify it works → mark ready for cutover
9. Launch CUTOVER instance → verify it works
10. Finalize cutover
11. Clean up everything
```

That's it — this is the full migration, start to finish.
