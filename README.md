# Search for a String and Save Output (RHEL 9)
### RHCSA EX200 Lab | Part of [linux-ops-mastery](https://github.com/kelvintechnical/linux-ops-mastery)
> Install and enable the Apache web server, search for a specific string in its config file, and save the output to a file under `/root`.

![RHCSA](https://img.shields.io/badge/RHCSA-EX200-EE0000?style=flat&logo=redhat&logoColor=white)
![Topic](https://img.shields.io/badge/Topic-File_Operations-blue)

---

## 📋 Scenario

On **Node1**, ensure the Apache web server is installed and running. Find all lines containing the string `"Listen"` in `/etc/httpd/conf/httpd.conf` and save the output to `/root/httpd_listen.txt`. Verify the file contains the correct matching lines.

## 🎯 Requirements

1. Install the `httpd` package if not already present.
2. Enable and start the `httpd` service.
3. Search for all lines containing `"Listen"` in `/etc/httpd/conf/httpd.conf`.
4. Save the output to `/root/httpd_listen.txt`.
5. Verify the file contains the correct lines.

## ✅ Tasks

- Install `httpd` via `dnf`
- Enable and start `httpd` via `systemctl`
- Use `grep` to search the config file
- Redirect output to `/root/httpd_listen.txt`
- Verify with `cat`

---

## 📚 Command Decision Map

| Lab Phrase | Question Being Asked | Tool |
|------------|---------------------|------|
| "Ensure installed" | How do I install a package? | `dnf install` |
| "Enabled and running" | How do I start + persist a service? | `systemctl enable --now` |
| "Find all lines containing" | How do I search a file for a string? | `grep "string" /path/to/file` |
| "Save the output" | How do I redirect output to a file? | `>` redirect operator |
| "Save under /root" | Who owns /root? | `sudo` required — root directory |
| "Verify correct lines" | How do I confirm file contents? | `cat /root/httpd_listen.txt` |

---

## Step 1 — Install Apache

```bash
sudo dnf install httpd -y
```

> **What this does:** Pulls `httpd` and its dependencies from your enabled repos. The `-y` flag auto-confirms the install prompt.

---

## Step 2 — Enable and start httpd

```bash
sudo systemctl enable --now httpd
```

> **What this does:** `--now` combines enable (persist at boot) and start (activate immediately) in one command.

**Confirm it's running:**
```bash
systemctl status httpd | grep Active
# Expected: Active: active (running)
```

---

## Step 3 — Search and save output

```bash
sudo grep "Listen" /etc/httpd/conf/httpd.conf | sudo tee /root/httpd_listen.txt
```

> **Why `tee` instead of `>`?** The `/root/` directory is root-owned. A plain `>` redirect runs as your user even with `sudo grep` — it fails on the write. `tee` runs as root and handles the write correctly.

---

## Step 4 — Verify the file

```bash
cat /root/httpd_listen.txt
```

**Expected output:**
