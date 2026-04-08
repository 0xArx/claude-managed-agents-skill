# Cloud Container Reference

Each session runs in an isolated Ubuntu 22.04 LTS container (x86_64).
Containers are ephemeral — destroyed after the session ends.
Deliverable files must be written to `/mnt/session/outputs/` to be retrievable via the Files API.

---

## Container Specs

| Resource | Limit |
|---|---|
| RAM | Up to 8 GB |
| Disk | Up to 10 GB |
| Architecture | x86_64 |
| OS | Ubuntu 22.04 LTS |

---

## Pre-Installed Language Runtimes

These are available in every container without any `apt` or `pip` install:

| Runtime | Version |
|---|---|
| Python | 3.12+ (with pip) |
| Node.js | 20+ (with npm) |
| Go | 1.22+ |
| Rust | 1.77+ (with cargo) |
| Java | 21+ (OpenJDK, with Maven and Gradle) |
| Ruby | 3.3+ (with gem) |
| PHP | 8.3+ (with composer) |
| C / C++ | GCC 13+ |

---

## Pre-Installed Databases

| Database | Status |
|---|---|
| SQLite | Ready to use — no install needed |
| PostgreSQL | Client only (`psql`) — no server running by default |
| Redis | Client only (`redis-cli`) — no server running by default |

To run a PostgreSQL or Redis server, install and start it via bash:
```bash
apt-get install -y postgresql && pg_ctlcluster 14 main start
```

---

## Pre-Installed System Utilities

All of these are available without installing:

```
git, curl, wget, jq, ripgrep (rg), unzip, zip, tar, ffmpeg,
imagemagick, pandoc, wkhtmltopdf, xvfb (virtual display),
chromium-browser (headless), build-essential, cmake
```

---

## Package Managers (for runtime installs)

The agent can install packages at runtime using bash. Supported in environment config:

| Manager | Language | Example |
|---|---|---|
| `pip` | Python | `pip install pandas` |
| `npm` | Node.js | `npm install axios` |
| `apt` | System | `apt-get install -y sqlite3` |
| `cargo` | Rust | `cargo add serde` |
| `gem` | Ruby | `gem install nokogiri` |
| `go` | Go | `go get github.com/...` |

Pre-declare heavy packages in the environment config so they're cached across sessions:
```python
environment = client.beta.environments.create(
    name="data-science-env",
    config={
        "type": "cloud",
        "networking": {"type": "unrestricted"},
        "packages": {
            "pip": ["pandas", "numpy", "matplotlib", "scikit-learn", "openpyxl"],
            "apt": ["libpq-dev"],
        },
    },
)
```

If using `limited` networking, add `"allow_package_managers": True` or runtime installs will fail.

---

## Filesystem Layout

```
/mnt/session/
├── outputs/    ← write deliverables here; fetchable via Files API after session
├── inputs/     ← mounted files from resources[] appear here (or your mount_path)
/repo/          ← GitHub repository mounted here (if resource defined)
/tmp/           ← scratch space, lost on session end
```

---

## Retrieving Output Files

Files written to `/mnt/session/outputs/` can be downloaded after the session:

```python
files = client.beta.files.list(scope_id=session.id)
for f in files.data:
    print(f"{f.id}: {f.filename} ({f.size_bytes} bytes)")

content = client.beta.files.download(files.data[0].id)
content.write_to_file("output.xlsx")
```
