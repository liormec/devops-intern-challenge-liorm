# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output
* Building the proxy image (`docker compose build proxy` or `./scripts/run-tests.sh`) failed at the `RUN apt-get update && apt-get install -y nginx ...` step.
* Log output showed:
  ```text
  Configuration file '/etc/nginx/nginx.conf'
   ==> File on system created by you or by a script.
   ==> File also in package provided by package maintainer.
     What would you like to do about it ?  Your options are:
  *** nginx.conf (Y/I/N/O/D/Z) [default=N] ? dpkg: error processing package nginx-common (--configure):
   end of file on stdin at conffile prompt
  ```

### How I diagnosed it
* Ran `./scripts/run-tests.sh` and inspected the complete build log rather than solely the final non-zero exit code.
* Identified that `dpkg` was halting at an interactive conffile prompt during the installation of `nginx-common`.

### Root cause
* The Dockerfile executed `COPY nginx.conf /etc/nginx/nginx.conf` before `apt-get install -y nginx`.
* `/etc/nginx/nginx.conf` is a package-managed configuration file (`conffile`).
* When `dpkg` unpacked `nginx-common`, it detected an untracked pre-existing configuration file and opened an interactive prompt on `stdin` asking whether to overwrite or keep it.
* Because Docker container builds run without an interactive TTY attached, `dpkg` encountered EOF on `stdin` and aborted with exit code 1.

### Fix
* Reordered instructions so `apt-get install` completed before copying the custom configuration file.
* (Further optimized in Bonus 1 by migrating to the minimal `nginx:alpine` image).

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
* The proxy container built and `/nginx-health` returned HTTP 200, but proxied requests to `/api/info` failed with `HTTP Error 502: Bad Gateway`.
* Nginx error log reported:
  ```text
  connect() failed (111: Connection refused) while connecting to upstream, upstream: "http://127.0.0.1:8080/api/info"
  ```

### How I diagnosed it
* Examined `docker compose logs proxy` and correlated the 502 response from `test-runner` with Nginx's connection failure.

### Root cause
* In Docker bridge networks, each container runs in its own network namespace with an isolated loopback interface (`127.0.0.1`).
* `proxy/nginx.conf` set the upstream server to `server 127.0.0.1:8080;`.
* Inside the proxy container, `127.0.0.1` refers to itself, where nothing was listening on port 8080, causing `ECONNREFUSED (111)`.

### Fix
* Changed `server 127.0.0.1:8080;` to `server backend:8080;` in `proxy/nginx.conf`.
* Leveraged Docker Compose's built-in DNS service discovery to route traffic directly to the backend container's IP.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
* Tests failed on `/api/info` with `HTTP Error 502: Bad Gateway` / `connect() failed (111: Connection refused)`.
* Backend container logs revealed that only the first configuration key (`[Backend] loaded config key: service`) had loaded before the test runner exited.

### How I diagnosed it
* Inspected `backend/app.py` and saw `load_config()` iterates over 4 keys with `time.sleep(CONFIG_FETCH_SECONDS)` (2 seconds per key = 8 seconds total startup delay).
* Observed that `test-runner` started immediately and issued requests within milliseconds, before the backend started listening on port 8080.

### Root cause
* Default `depends_on` in Docker Compose only tracks container lifecycle (process started), not application readiness.
* `test-runner` executed tests before the backend finished its 8-second startup delay.
* Per challenge ground rules, `backend/app.py` could not be modified and tests could not be weakened.

### Fix, and why I chose it over the alternatives
* Added a native Docker `healthcheck` to the backend service targeting `http://localhost:8080/health`.
* Configured `proxy` and `test-runner` to depend on `backend` with `condition: service_healthy`.
* **Why chosen:** Healthcheck orchestration is deterministic, production-grade, and avoids crude shell sleep hacks or application code modifications.

---

## Defect 4: CI pipeline fails

### Symptom / error output
* Running `./scripts/run-tests.sh` failed with:
  ```text
  zsh: permission denied: ./run-tests.sh
  ```
* In GitHub Actions (`.github/workflows/ci.yml`), step `Execute test suite` fails with `Permission denied`.

### How I diagnosed it
* Checked file mode bits with `ls -l scripts/run-tests.sh` and saw `-rw-r--r--` (no execute `x` bit).

### Root cause
* The shell script was checked into Git without executable file permissions.
* GitHub Actions runner checks out the repository and retains Git's file permissions, causing direct invocation `./scripts/run-tests.sh` to fail.

### Fix
* Granted execution permissions locally: `chmod +x scripts/run-tests.sh`.
* Tracked the executable bit in Git's index: `git update-index --chmod=+x scripts/run-tests.sh`.

---

## Bonus questions

### 1. Minimal Base Image & Size Reduction
* Switched `proxy/Dockerfile` from `ubuntu:24.04` to `nginx:alpine`.
* Removed package installations (`apt-get`) and debugging tools (`htop`, `vim`, `git`, `curl`).
* **Image size before:** 325 MB
* **Image size after:** 68.5 MB (~79% reduction).
* **Build time:** Reduced from ~55s to ~1.7s (~97% faster).

### 2. Running as Non-Root User
* Configured `proxy/Dockerfile` to run as unprivileged user `USER nginx` (UID 101).
* Added `pid /tmp/nginx.pid;` in `proxy/nginx.conf` so Nginx can write its process ID without root permissions on `/var/run`.
* Set directory ownership on `/var/cache/nginx` and `/tmp/nginx.pid` (`chown -R nginx:nginx`).
* Verified tests pass and proxy runs as an unprivileged process.

---

## Anything you would do differently with more time
* Implement a multi-stage Dockerfile or distroless image for the proxy to reduce attack surface even further.
* Add pre-commit hooks running `shellcheck` for shell scripts and `hadolint` for Dockerfiles.
* Mount container root filesystems as read-only (`read_only: true` with tmpfs mounts) for enhanced defense-in-depth security.
