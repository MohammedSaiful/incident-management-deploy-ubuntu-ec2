# Incident Management App: Manual Deployment on Ubuntu EC2

A three-tier incident management application deployed **manually** on an AWS EC2 instance running **Ubuntu Server 24.04 LTS**. It uses Nginx as a reverse proxy and systemd to keep the backend running, with **no Docker or Docker Compose**.

> Part of the **bongoDev** DevOps lab series. The goal is to understand what each server component does before automating deployments with GitHub Actions.

## Tech Stack

| Layer | Technology |
| --- | --- |
| Frontend | React + Vite |
| Backend | Node.js 24 + Express REST API |
| Database | PostgreSQL 16 |
| Web server / reverse proxy | Nginx |
| Process manager | systemd |
| Hosting | AWS EC2 (Ubuntu 24.04 LTS, t3.small) |

## Architecture

```text
Your browser
    |
    | HTTP :80
    v
Ubuntu EC2: Nginx
    |-- / and static assets ---> React production files (/var/www/incident-management)
    |
    `-- /api/* and /health ---> Express / Node.js (127.0.0.1:5000)
                                    |
                                    `-- PostgreSQL (127.0.0.1:5432)

systemd keeps the Node.js backend service alive.
```

Only **port 80** (HTTP) is open to the Internet, and **port 22** (SSH) is restricted to my own IP. Ports 5000 (backend) and 5432 (PostgreSQL) are never exposed publicly.

## Repository Structure

```text
.
├── README.md
├── frontend/          # React + Vite app
├── backend/           # Express REST API
├── scripts/           # Install, database setup, deploy and verify scripts
├── deploy/            # Nginx config and systemd service files
├── docs/              # Extra documentation (GitHub Actions lesson)
└── .github/workflows/ # Optional deployment workflow
```

## Deployment Steps

### 1. Launch EC2
- Ubuntu Server 24.04 LTS (64-bit x86), `t3.small`, 20 GiB gp3 storage.
- Security group: SSH (22) from my IP only, HTTP (80) from anywhere.
- SSH key pair created and kept private (never committed).

### 2. Connect and copy the project

```bash
chmod 400 bongodev-ec2.pem
ssh -i bongodev-ec2.pem ubuntu@EC2_PUBLIC_IP
scp -i bongodev-ec2.pem incident-management-ubuntu-ec2.zip ubuntu@EC2_PUBLIC_IP:/home/ubuntu/
```

### 3. Install Node.js, PostgreSQL and Nginx

```bash
bash scripts/install-nodejs-ubuntu.sh
bash scripts/install-postgresql-ubuntu.sh
bash scripts/install-nginx-ubuntu.sh
sudo apt-get install -y git curl openssl rsync
```

### 4. Create the database and application user

```bash
bash scripts/setup-database.sh
```

This creates the `incident_user` user and the `incidentdb` database, then writes the credentials to `backend/.env` with restrictive permissions. That file is git-ignored and must never be committed.

### 5. Deploy the application

```bash
bash scripts/deploy-ubuntu.sh
```

The script:
1. Installs frontend and backend dependencies.
2. Builds the React production bundle with `npm run build`.
3. Copies the build to `/var/www/incident-management/`.
4. Configures and restarts the `incident-backend` systemd service on `127.0.0.1:5000`.
5. Configures Nginx to serve the UI and proxy `/api/` and `/health` to Express.
6. Checks that the backend reports a healthy database connection.

## Verification

```bash
bash scripts/verify.sh http://127.0.0.1
bash scripts/check-services.sh
sudo systemctl status incident-backend --no-pager
sudo nginx -t
curl http://127.0.0.1/health
curl http://127.0.0.1/api/incidents
```

- `/health` returns `"status":"healthy"` and `"database":"connected"`.
- The app opens in a browser at `http://EC2_PUBLIC_IP/` (HTTP, no TLS in this lab).
- Creating an incident and changing its status persists after a page refresh.
- Data can be confirmed directly in PostgreSQL:

```bash
sudo -u postgres psql -d incidentdb -c 'SELECT id, title, severity, status FROM incidents ORDER BY id DESC;'
```

## Manual Redeployment

To prove the app can be updated, the heading in `frontend/src/main.jsx` was changed from `Incident Command Center` to `Incident Command Center — Updated Manually`, and then the deployment script was run again:

```bash
bash scripts/deploy-ubuntu.sh
```

The new heading appeared in the browser, and the existing incidents were kept because the script does not drop the database.

## Screenshots

Add your screenshots to a `docs/screenshots/` folder and reference them here:

```markdown
![EC2 instance running](images/01_EC2_Running.jpg)
![project transfer to ec2](images/02_project_transfer_to_ec2.jpg)
![project tree](images/03_project_tree.jpg)
![Installed node versions](images/04_node_version.jpg)
![Installed psql versions](images/05_psql_version.jpg)
![Installed ngnix versions](images/06_ngnix_version.jpg)
![open using ec2 ip](images/07_open_using_ec2_ip.jpg)
![created new incident](images/08_created_new_incident.jpg)

```

Make sure no passwords, `.env` contents, SSH keys or sensitive account details are visible in any screenshot.

## How the Components Work Together

1. The **browser** sends an HTTP request to the EC2 public IP on port 80.
2. **Nginx** serves the React static files for `/`, and forwards `/api/*` and `/health` to the Express backend.
3. **Express** (Node.js) handles the REST API on `127.0.0.1:5000` and talks to the database.
4. **PostgreSQL** stores incidents so they survive page refreshes and backend restarts.
5. **systemd** keeps the backend running and restarts it if it crashes.

## Answers to Extra Credit Questions

**1. Why is PostgreSQL not exposed on public port 5432?**
Exposing a database to the Internet invites brute-force and exploit attempts. Only the backend on the same machine needs it, so it listens on `127.0.0.1` and the security group blocks the port.

**2. What is the difference between a systemd service and an Nginx reverse proxy?**
systemd starts, supervises and restarts processes (here, the Node.js backend). Nginx receives web traffic, serves static files and forwards API requests to the backend. One manages the process, the other manages the traffic.

**3. Why do we build React with `npm run build` before deploying?**
The build compiles and bundles the JSX and modules into optimized static HTML, JS and CSS that browsers can run. Nginx can then serve these files directly, with no Node or Vite on the frontend at runtime.

**4. Why does changing React code require a build/redeploy, but creating an incident does not?**
React code is compiled into static files, so changes only appear after a rebuild and a copy to the web root. Creating an incident is just data stored in PostgreSQL through the API, and it needs no code change.

**5. Which actions could GitHub Actions perform automatically after `git push`?**
Check out the code, install dependencies, run tests, build the frontend, connect to the EC2 instance securely, copy the files, run `deploy-ubuntu.sh`, restart services and verify the health check.

## Next Step: Automation

The same `deploy-ubuntu.sh` script can be run by a GitHub Actions workflow (see `docs/GITHUB_ACTIONS.md`), so every `git push` redeploys the app automatically. The workflow deploys to this existing Ubuntu instance and does not create the EC2 server.

## Cleanup

To avoid AWS charges, after capturing the evidence:

1. Remove any demo secrets and SSH deployment keys from GitHub.
2. Terminate the `bongodev-incident-ubuntu` instance in **EC2 > Instances**.
3. Check that the EBS volume and any Elastic IPs are released.

The demo database is disposable and is lost when the storage is deleted.

## Security Notes

- Never commit `backend/.env`, passwords or SSH private keys.
- Keep ports 5000 and 5432 closed to the Internet, and restrict SSH to your own IP.
- This demo uses HTTP and has no login. It is for learning only and is not production-ready, so do not enter real personal or company data.

---



# Lab Answers: Manual Deployment of Incident Management on Ubuntu EC2
---
## Part 1:

### 1. Why is PostgreSQL not exposed on public port 5432?

- A database open to the Internet can be attacked with brute-force logins and exploits.
- Only the backend on the same server needs the database, so PostgreSQL listens on `127.0.0.1` (localhost) only.
- The EC2 security group also keeps port 5432 closed.
- This follows the principle of least exposure: only Nginx on port 80 is public (and SSH on port 22 from my own IP).

### 2. What is the difference between an Ubuntu service (`systemd`) and an Nginx reverse proxy?

| | systemd service | Nginx reverse proxy |
| --- | --- | --- |
| Job | Starts, supervises and restarts processes | Receives web traffic and routes it |
| In this lab | Keeps the Node.js backend (`incident-backend`) running | Serves the React files and forwards `/api` and `/health` to Express |
| Handles | Process lifecycle (boot start, crash restart, logs) | HTTP requests (static files, proxying) |

One manages the **process**, the other manages the **traffic**.

### 3. Why do we build React using `npm run build` before deploying?

- Browsers cannot run JSX and Vite source files directly.
- The build compiles and bundles the code into optimized static HTML, JavaScript and CSS files.
- Nginx can serve these files directly from `/var/www/incident-management`, so no Node.js or Vite process is needed for the frontend at runtime.
- The production bundle is smaller and faster than the development version.

### 4. Why does changing React code require a build/redeploy, but creating an incident does not?

- React code is compiled into static files, so a code change only appears after rebuilding and copying the new files to the web root (what `deploy-ubuntu.sh` does).
- Creating an incident is just **data**. The browser sends it to the API, and Express saves it in PostgreSQL. No code changes, so no rebuild is needed.

### 5. Which actions could GitHub Actions perform automatically after `git push`?

- Check out the repository.
- Install dependencies and run tests or linting.
- Build the React frontend.
- Connect to the EC2 instance securely (for example over SSH with a deployment key stored in GitHub Secrets).
- Copy the updated files to the server.
- Run `scripts/deploy-ubuntu.sh` to rebuild, restart the service and reload Nginx.
- Run the health check (`/health`) to verify the deployment.

---

## Part 2:

### If the backend crashes, what restarts it?

**systemd.** The `incident-backend.service` unit is configured to restart the Node.js process if it exits, and to start it at boot. You can check it with `sudo systemctl status incident-backend` and view logs with `sudo journalctl -u incident-backend`.

### Why does the backend bind to `127.0.0.1:5000`?

- Binding to localhost means the backend only accepts connections from the same machine.
- Users cannot reach Express directly. All public traffic must pass through Nginx on port 80.
- This keeps the backend off the public Internet, and Nginx becomes the single entry point.

### Why is PostgreSQL not accessible from the Internet?

- It listens only on `127.0.0.1:5432`, and the security group does not allow port 5432.
- Only the backend on the same server connects to it, using a dedicated user (`incident_user`) with a generated password stored in the git-ignored `backend/.env`.

---

## Part 3: Short Explanation for Submission

**How Browser → Nginx → Express → PostgreSQL work together, and which steps GitHub Actions could automate**

The browser sends an HTTP request to the EC2 public IP on port 80. Nginx receives it. For `/` and static assets, it serves the compiled React files from `/var/www/incident-management`. For `/api/*` and `/health`, it forwards the request to the Express backend on `127.0.0.1:5000`. Express runs the business logic and reads or writes incidents in PostgreSQL, which stores the data so it survives page refreshes and restarts. systemd keeps the backend alive and restarts it if it crashes. GitHub Actions could automate the manual steps: checking out the code, installing dependencies, building the React app, copying files to EC2 and running `deploy-ubuntu.sh` after every `git push`, then verifying the health check.
