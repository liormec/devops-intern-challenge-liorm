# Root Cause Analysis

---

## Defect 1: Proxy image fails to build

### Symptom / error output
* Running `./scripts/run-tests.sh` stopped during `docker compose build proxy`.
* In the middle of the build log, `dpkg` printed:
  ```text
  Configuration file '/etc/nginx/nginx.conf'
   ==> File on system created by you or by a script.
   ==> File also in package provided by package maintainer.
     What would you like to do about it ?  Your options are:
  *** nginx.conf (Y/I/N/O/D/Z) [default=N] ? dpkg: error processing package nginx-common (--configure):
   end of file on stdin at conffile prompt
  ```

### How I diagnosed it
* Scrolled up the build logs to find where it actually got stuck, rather than just reading the error code at the very bottom.

### Root cause
* The Dockerfile had `COPY nginx.conf /etc/nginx/nginx.conf` before `apt-get install -y nginx`.
* When Ubuntu installs Nginx, `dpkg` sees a custom configuration file already sitting there and pauses to ask the user on the keyboard whether to overwrite it.
* Since Docker build doesn't have an interactive terminal, `dpkg` received an end-of-file (EOF) and crashed.

### Fix
* Change order of the lines so `apt-get install` finishes first, and our custom `nginx.conf` is copied after.

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
* The proxy built and `/nginx-health` worked, but requests to `/api/info` failed with `502 Bad Gateway`.
* Nginx logs showed:
  ```text
  connect() failed (111: Connection refused) while connecting to upstream: "http://127.0.0.1:8080/api/info"
  ```

### How I diagnosed it
* Looked at `docker compose logs proxy` to see what Nginx was complaining about.

### Root cause
* `proxy/nginx.conf` pointed to `127.0.0.1:8080`.
* In Docker, each container has its own localhost (`127.0.0.1`). The proxy was trying to connect to port 8080 on itself instead of the backend container.

### Fix
* Changed `127.0.0.1:8080` to `backend:8080` in `nginx.conf`, so Nginx uses Docker's internal DNS to reach the backend service.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
* Tests crashed with `502 Bad Gateway` right away.
* Looking at the backend logs, it had only loaded the first config key (`[Backend] loaded config key: service`) before the test runner gave up.

### How I diagnosed it
* Checked `backend/app.py` and saw `load_config()` sleeps 2 seconds for each of the 4 keys (total of 8 seconds) before it starts listening on port 8080.
* Looked at timestamps and saw `test-runner` was sending requests immediately when containers started, way before the backend was ready.

### Root cause
* By default, `depends_on` in Docker Compose only waits for the container process to start, not for the app inside it to actually finish booting.
* `test-runner` ran too fast and hit a closed port. We weren't allowed to edit `app.py` or weaken the tests.

### Fix, and why I chose it over the alternatives
* Added a healthcheck to `backend` in `docker-compose.yml` polling `/health`, and made `proxy` and `test-runner` wait for `condition: service_healthy`.
* **Why this fix:** It's the standard Docker Compose way to handle startup timing without adding hacky `sleep` scripts or changing application code.

---

## Defect 4: CI pipeline fails

### Symptom / error output
* Running `./scripts/run-tests.sh` gave `zsh: permission denied: ./run-tests.sh`.

### How I diagnosed it
* Checked permissions using `ls -l scripts/run-tests.sh` and saw `-rw-r--r--`.

### Root cause
* The script didn't have execute permissions (`+x`). Even if I ran `bash scripts/run-tests.sh` locally, GitHub Actions CI runs `./scripts/run-tests.sh` directly on Ubuntu and fails with permission denied.

### Fix
* Made it executable with `chmod +x scripts/run-tests.sh`.
* Told Git to save the execute bit with `git update-index --chmod=+x scripts/run-tests.sh`.

---

## Bonus questions

### 1. Minimal Base Image
* Switched the proxy from `ubuntu:24.04` to `nginx:alpine`.
* Removed packages we don't need in production like `vim`, `git`, and `htop`.
* **Image size before:** 325 MB
* **Image size after:** 68.5 MB (~79% smaller!)
* **Build time:** Went from ~55 seconds down to ~2 seconds.

### 2. Running as Non-Root User
* Added `USER nginx` in `proxy/Dockerfile`.
* Added `pid /tmp/nginx.pid;` in `nginx.conf` so Nginx writes its PID file to `/tmp` instead of `/var/run` (which requires root).
* Gave the `nginx` user ownership of `/var/cache/nginx` and `/tmp/nginx.pid`.

---

## Anything you would do differently with more time
* Make the proxy filesystem read-only (`read_only: true`) with tmpfs mounts for extra security.
