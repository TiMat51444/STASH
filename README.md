# STASH — Always Synced

STASH is a desktop note and file-sharing system built in Python using standard TCP sockets and Tkinter. It pairs a desktop client (`client.py`) with two replicated file servers (**Primary** and **Backup** via `server.py`) so users can upload, browse, preview, download, rename, and delete shared note files with automatic failover and peer synchronization.

---

## Features

* **Dual-Server Failover & Sync:** Runs two server nodes (`Primary` on port `8000` and `Backup` on port `8001`). Mutations (uploads, renames, deletes, account creation, and user deletion) are replicated to the peer server, and a server that restarts catches up automatically from its peer.
* **Automatic Client Failover & Health Monitoring:** The client probes both servers every 3 seconds and automatically switches to the healthy server if the active one goes down, displaying banners and toast notifications without dropping the user's session.
* **User Accounts & Role Permissions:** Supports user registration, sign-in, and role-based access control (`user` vs. `admin`). Users can rename or delete files they own; admins can manage any file and remove non-primary-admin user accounts.
* **File Organization & In-App Preview:** Filter files by *All files*, *My uploads*, or type groups (*Text notes*, *PDFs*, *Documents*, *Slides*, *Images*, *Spreadsheets*), search by filename or owner, sort columns, and preview `.txt` and image files directly inside the details pane.
* **File Constraints:** Supports `.txt`, `.pdf`, `.doc`, `.docx`, `.ppt`, `.pptx`, `.png`, `.jpg`, `.jpeg`, `.xls`, and `.xlsx` files up to **50 MB** each.

---

## Project Structure

```text
.
├── client.py          # Tkinter desktop GUI, cluster client, background health monitor
├── server.py          # TCP file & user server with peer replication and startup sync
├── logo.png           # Optional custom logo (falls back to built-in badge if absent)
├── logo_84.png        # Optional 84px logo when Pillow is not installed
├── logo_32.png        # Optional 32px logo when Pillow is not installed
├── client_error.log   # Generated automatically if unhandled client errors occur
└── server_data_<port>/
    ├── storage/       # Stored note/file binaries
    ├── users.json     # User credentials and roles
    └── metadata.json  # File ownership mapping
```

---

## Requirements

* **Python 3.x** with `tkinter` (standard library).
  
---

## Getting Started

### 1. Start the Primary and Backup Servers

Open two terminals and run both server instances on `127.0.0.1`:

```bash
# Terminal 1 — Primary server (port 8000)
python server.py primary
```

```bash
# Terminal 2 — Backup server (port 8001)
python server.py backup
```

On first launch, each server initializes its `server_data_<port>` directory with a default administrator account:

* **Username:** `admin`
* **Password:** `adminpassword`

### 2. Launch the Desktop Client

In a third terminal, start the GUI app:

```bash
python client.py
```

From the login screen, you can sign in with an existing account or click **Create an account** to register a new user (note: usernames cannot contain `:`, and passwords must be at least 4 characters).

---

## Keyboard Shortcuts

| Shortcut | Action |
| :--- | :--- |
| `Ctrl + U` | Upload files |
| `Ctrl + F` | Focus the search bar |
| `F5` | Refresh the file list |
| `F2` | Rename the selected file (if permitted) |
| `Delete` | Delete the selected file (when not typing in an input field) |

---

## How It Works

1. **Framing Protocol:** Both `client.py` and `server.py` send messages over TCP prefixed with a 4-byte big-endian unsigned integer (`>I`) indicating payload length, followed by the raw payload bytes.
2. **Cluster Routing:** `Cluster.request()` tries the currently active node first. If it fails, it logs a `crash` event, tries the other node, updates `self.active` on success (`failover`), or reports `down` if both are unreachable.
3. **Peer Replication & Catch-Up:** When a client writes, renames, or deletes a file—or registers or deletes a user—the handling server applies the change under a threading lock and forwards a `SYNC_*` command to its peer. When a server starts up, `sync_from_peer()` pulls users, metadata, and files from the peer and removes stale local files not present in peer metadata.

---

## Notes & Limitations

* **Localhost Configuration:** `NODES` in `client.py` and `HOST` / port arguments in `server.py` default to `127.0.0.1` (`8000` and `8001`).
* **Authentication & Storage:** Passwords are stored in plain text inside `users.json` and traffic is sent over unencrypted TCP sockets—intended for local/educational or trusted environments rather than untrusted networks.
