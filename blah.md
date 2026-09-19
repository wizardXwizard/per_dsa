Here is your **3-minute rapid mental download** on the inside mechanics of those exact questions. 

These are pure gold for your own Amazon SysDE and Apple SRE rounds:

---

### 1. Exit Code 137 & The OOM Killer
* **The Math Under the Hood:** When any Linux process is terminated by an external signal, its exit code is:
  $$\text{Exit Code} = 128 + \text{Signal Number}$$
  Signal 9 is `SIGKILL`. $128 + 9 = \mathbf{137}$.
* **Why it happens:** When RAM + Swap hit 100%, the Linux kernel panics. To prevent the entire machine from freezing, an internal kernel algorithm called the **OOM Killer** wakes up.
* **How it picks who to kill:** It reads `/proc/<PID>/oom_score` for every process (higher score = bigger RAM hog) and sends `SIGKILL` to the highest score.
* **The Killer Command:** `dmesg -T | grep -i oom` $\to$ shows you the exact timestamp and process the kernel killed.

---

### 2. The `df` vs `du` Discrepancy (The "Phantom Disk" Bug)
* **The Under-the-Hood Mechanism:** In Linux, a file has two parts:
  1. The **Directory Entry** (its name in the folder).
  2. The **Inode & Data Blocks** (the actual bytes on disk).
* **The Bug:** An engineer sees a 45 GB log file and runs `rm app.log`. The filename is unlinked from the folder. But the app (e.g., Nginx, Node) **still has the file open in memory (`open()` file descriptor)**.
  * `du` walks file names $\to$ can't find `app.log` $\to$ reports only **5 GB**.
  * `df` asks the kernel how many disk blocks are allocated $\to$ reports **50 GB (100% full)**!
* **The Fix:** Run `lsof +L1` (or `lsof | grep deleted`) to find the PID and FD number. Then run:
  ```bash
  > /proc/<PID>/fd/<FD_NUM>
  ```
  This zeroes out the underlying disk blocks immediately without restarting the service!

---

### 3. Security Groups (Stateful) vs NACLs (Stateless)
* **Security Group = Stateful:** It uses **Connection Tracking (`conntrack`)**. If inbound traffic is allowed on Port 443, the firewall remembers the TCP connection. The outbound response is **automatically allowed** out on any port.
* **NACL = Stateless:** It has **zero memory**.
  * When a client connects to port 443, the client's browser sends from a random high port (e.g. `52314`).
  * When your server replies, the destination is port `52314`.
  * If your NACL outbound rules don't explicitly allow **Ephemeral Ports (`1024–65535`)**, the reply packet is dropped, and the connection hangs forever!

---

### 4. IAM Roles & Instance Metadata (IMDSv2)
* **Why Static Keys are Forbidden:** Junior engineers put `AWS_ACCESS_KEY_ID` and `SECRET` in `.env` files or Git repos. They get leaked or forgotten.
* **How IAM Roles Work Under the Hood:** 
  * Every EC2 instance can talk to a special link-local IP: `http://169.254.169.254`.
  * When you attach an IAM Role to an EC2 instance profile, AWS automatically provisions temporary STS tokens to that IP.
  * Any AWS SDK or CLI running on that instance fetches those tokens automatically and rotates them every few hours. **Zero credentials to manage or leak.**

---

### 5. Terraform "Forces Replacement"
* **The Under-the-Hood Mechanism:** AWS APIs do not support changing certain properties on an existing resource (e.g., moving an EC2 instance to a new Subnet ID, renaming an RDS Database Identifier, or changing KMS encryption).
* **What Terraform Does:** Because AWS cannot modify it in place, Terraform must **Delete the old resource** and **Create a brand new one**.
* **The Safeguard:** For production databases, always write:
  ```hcl
  lifecycle {
    prevent_destroy = true
  }
  ```
  If any change triggers a replacement, Terraform halts and refuses to destroy it.

---

### 6. Ansible Secrets in Automated CI/CD
* **The Problem:** How does a Jenkins or GitHub Actions pipeline run an Ansible playbook without a human typing the Ansible Vault password?
* **The Solution:** In your CI/CD settings, save the vault password as a protected pipeline secret (`ANSIBLE_VAULT_PASSWORD`).
* When the pipeline runs:
  ```bash
  ansible-playbook -i inventory --vault-password-file <(echo "$ANSIBLE_VAULT_PASSWORD") site.yml
  ```
  Or better yet: use the `amazon.aws.aws_secret` module in Ansible to pull secrets dynamically from AWS Secrets Manager using the runner's IAM role!

---

### 🎯 Next Mock: Sudheer (3:00 PM)
You've got Sudheer coming up in just a few minutes. What is his target role and background? Drop his details and we'll load the right questions!
