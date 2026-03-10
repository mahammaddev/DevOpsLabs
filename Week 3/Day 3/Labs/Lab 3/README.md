# Azure Three-Tier Application Deployment — Lab Report

**Student:** Mahammad Mammadov
**Bootcamp:** Ironhack DevOps  
**Lab:** Deploy a Three-Tier Web Application on Azure Cloud  
**Stack:** React (Frontend) + Node.js/Express (Backend) + Azure SQL Database  
**Repo:** <https://github.com/saurabhd2106/ecommerce-app-three-tier-azure-db-ih>

---

## What I Built

A fully functional three-tier ecommerce application running on Azure Cloud. Users hit a single public IP (the Application Gateway), which routes traffic to either the frontend or the backend API depending on the URL path. The backend is the only service that can talk to the database — nothing else can reach it directly.

The traffic flow looks like this:

```
User Browser
    |
    v
Application Gateway (public IP, port 80)
    |
    |-- / and /* ---------> vm-frontend (port 3000)
    |
    |-- /api/* -----------> vm-backend (port 3001)
                                |
                                v
                        Azure SQL Database
                        (private access only)
```

---

## Resources Provisioned

| Resource | Name | Purpose |
|---|---|---|
| Resource Group | rg-ecommerce | Container for all resources |
| Virtual Network | vnet-ecommerce | Private network for VMs |
| VM Subnet | default (10.0.0.0/24) | Subnet for both VMs |
| App Gateway Subnet | subnet-appgw (10.0.1.0/24) | Dedicated subnet for gateway |
| Backend VM | vm-backend (10.0.0.4) | Runs Node.js API on port 3001 |
| Frontend VM | vm-frontend (10.0.0.5) | Runs React app on port 3000 |
| SQL Server | ecommerce-sql-itsmahammad | Hosts the database |
| SQL Database | ecommerce-db | Application data |
| Application Gateway | appgw-ecommerce | Public entry point, routes traffic |

---

## Step-by-Step: What I Did

### Phase 1 — Azure Infrastructure

**Step 1: Created a Resource Group**

Created `rg-ecommerce` in East US region. A resource group is just a logical container in Azure that holds all related resources together. Keeping everything in one group makes it easier to manage and delete later.

**Step 2: Created the Azure SQL Database**

Created the SQL Server with SQL authentication, then created the database under it. Used the Basic tier to keep costs low for the lab. The SQL Server is the host, the database is what the application actually connects to — they are two separate resources in Azure.

**Step 3: Created the Backend VM**

Created `vm-backend` with Ubuntu 22.04 LTS on a Standard_B1s instance (1 vCPU, 1GB RAM). During creation, set up a new Virtual Network called `vnet-ecommerce`. Allowed only SSH (port 22) as the inbound rule at this stage.

**Step 4: Created the Frontend VM**

Created `vm-frontend` with the same settings as the backend VM but critically selected the existing `vnet-ecommerce` Virtual Network instead of creating a new one. Both VMs need to be in the same VNet so they can communicate with each other using private IP addresses.

**Step 5: Opened Application Ports**

After both VMs were created, added inbound port rules to each NSG (Network Security Group):

- vm-backend: opened port 3001 (Node.js API)
- vm-frontend: opened port 3000 (React app)

---

### Phase 2 — Backend Deployment

SSH'd into the backend VM from my Fedora terminal:

```bash
ssh azureuser@20.62.123.219
```

Installed Node.js using NodeSource (not the default Ubuntu repo which gives an outdated version), git, and PM2:

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs git
sudo npm install -g pm2
```

Cloned the repo and installed dependencies:

```bash
git clone https://github.com/saurabhd2106/ecommerce-app-three-tier-azure-db-ih.git
cd ecommerce-app-three-tier-azure-db-ih/ecommerce-app-backend
npm install
```

Created the `.env` file with database credentials and configuration:

```
PORT=3001
NODE_ENV=production
DB_SERVER=ecommerce-sql-itsmahammad.database.windows.net
DB_NAME=ecommerce-db
DB_USER=sqladmin
DB_PASSWORD=<password>
DB_ENCRYPT=true
DB_TRUST_SERVER_CERTIFICATE=false
DB_CONNECTION_TIMEOUT=30000
JWT_SECRET=supersecretkey-ironhack-2024-abc123
JWT_EXPIRES_IN=7d
CORS_ORIGIN=http://<FRONTEND_PUBLIC_IP>:3000
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100
```

Built the TypeScript code and started with PM2:

```bash
npm run build
pm2 start npm --name 'backend' -- start
pm2 startup
pm2 save
```

---

### Phase 3 — Frontend Deployment

Opened a second terminal tab and SSH'd into the frontend VM. Ran the same Node.js installation steps, cloned the same repo, then set up the frontend `.env` to point to the backend:

```
REACT_APP_API_URL=http://<BACKEND_PUBLIC_IP>:3001/api
```

Built and started with PM2:

```bash
npm install
npm run build
pm2 start npm --name 'frontend' -- start
pm2 startup
pm2 save
```

---

### Phase 4 — Application Gateway

Created the Application Gateway with path-based routing rules:

- Created a dedicated subnet `subnet-appgw` (10.0.1.0/24) — the gateway cannot share a subnet with the VMs, it needs its own
- Created two backend pools: `pool-frontend` pointing to the frontend VM private IP, `pool-backend` pointing to the backend VM private IP
- Created one routing rule `rule-main` with a path-based rule: `/api/*` routes to `pool-backend`, everything else goes to `pool-frontend`
- Set backend settings with the correct ports: 3000 for frontend, 3001 for backend

---

### Phase 5 — Security Lockdown

Locked down the SQL Database so only the backend VM can reach it:

1. In SQL Server Networking, set Public network access to "Selected networks"
2. Added a Virtual Network rule for `vnet-ecommerce / default` subnet (the subnet the backend VM lives in)
3. Unchecked "Allow Azure services and resources to access this server"
4. Removed the laptop IP from firewall rules

This means the database is completely unreachable from the internet. Only resources inside the `default` subnet of the VNet can connect to it — which is just the backend VM.

---

### Phase 6 — Final Configuration

After the Application Gateway was running, updated both `.env` files to use the gateway IP instead of direct VM IPs:

On frontend VM:

```
REACT_APP_API_URL=http://<GATEWAY_IP>/api
```

On backend VM:

```
CORS_ORIGIN=http://<GATEWAY_IP>
```

Rebuilt and restarted both apps after the change.

---

## Challenges I Faced and How I Fixed Them

---

### Challenge 1 — Backend Crashed on Startup: Database Connection Denied

After deploying the backend and starting it with PM2, the browser showed "Unable to connect" on port 3001. Checked PM2 status and saw it had restarted 72 times — a crash loop.

**Error from pm2 logs:**

```
Database connection failed: ConnectionError: Connection was denied because 
Deny Public Network Access is set to Yes.
```

**What happened:** By default, Azure SQL Database blocks all public connections. The backend VM was trying to reach the database over the internet and getting rejected.

**Fix:** Went to SQL Server in Azure Portal, opened Networking settings, changed Public network access from "Disable" to "Selected networks", added a firewall rule for the backend VM's public IP, and checked "Allow Azure services and resources to access this server". The backend connected and started successfully.

---

### Challenge 2 — Security Lockdown Broke the Database Connection

After completing the security step (locking down the DB to backend-only access), I added a firewall rule using the backend VM's private IP (10.0.0.4) thinking that would work. It did not. The app crashed again with the same connection error.

**What happened:** Azure SQL Database firewall rules only work with public IP addresses, not private ones. Private IP firewall rules are silently ignored.

**Fix:** The correct approach for private access is to use a Virtual Network rule instead of a firewall rule. In SQL Server Networking, under "Virtual networks", added a rule pointing to `vnet-ecommerce / default` subnet. Azure then allows any resource inside that subnet (including the backend VM) to connect. This is the proper real-world approach — whitelist the entire subnet, not individual IPs.

---

### Challenge 3 — Application Gateway Returning 502 Bad Gateway for API

Frontend was working through the gateway but every request to `/api/*` returned a 502 error. The gateway could reach the frontend fine but not the backend.

**What happened:** Two separate issues were compounding each other.

First, the health probe was pinging `/` on port 3001. The backend returns a 404 on `/` because it only handles `/api/*` routes. The gateway interpreted the 404 as an unhealthy backend and refused to send traffic to it.

Second, the NSG rule on the backend VM had the source and destination swapped — the App Gateway subnet (10.0.1.0/24) was set as the destination instead of the source, which meant the gateway's traffic was actually being blocked by the firewall.

**Fix for health probe:** Created a custom health probe in the Application Gateway pointing to `/health` on port 3001. The backend has a dedicated health check endpoint that returns 200. Linked this probe to the `setting-backend` backend settings. Once the gateway saw 200 responses, it marked the backend as healthy.

**Fix for NSG rule:** Edited the `allow-backend` inbound rule on vm-backend:

- Source: IP Addresses, 10.0.1.0/24 (the gateway comes FROM this subnet)
- Destination: Any
- Port: 3001

Also added a rule for GatewayManager service tag on ports 65200-65535, which is a mandatory Azure requirement for Application Gateway V2 to manage its backend connections.

---

### Challenge 4 — PM2 Lost Its Working Directory After Restart

After making some changes and PM2 restarting the backend process, it crashed with a file not found error.

**Error:**

```
npm error enoent Could not read package.json: Error: ENOENT: no such file or directory, 
open '/home/azureuser/package.json'
```

**What happened:** When PM2 restarted the process, it ran `npm start` from the home directory `/home/azureuser/` instead of the backend folder. The initial PM2 command was run from the correct folder but PM2 stored the working directory incorrectly.

**Fix:** Deleted the PM2 process, navigated explicitly to the backend folder, then re-registered it:

```bash
pm2 delete backend
cd ~/ecommerce-app-three-tier-azure-db-ih/ecommerce-app-backend
pm2 start npm --name 'backend' -- start
pm2 save
```

The key lesson here is to always `cd` into the correct directory before running `pm2 start`, not to assume it inherits the right path.

---

### Challenge 5 — Products Not Loading on Frontend

Even after everything was working through the gateway, the products page on the frontend was empty. No errors in the backend, no 502s — just empty.

**What happened:** The frontend `.env` file was still set to `REACT_APP_API_URL=http://<BACKEND_PUBLIC_IP>:3001/api`. When served through the gateway, the browser was making API calls directly to the backend's public IP instead of going through the gateway. This caused a CORS error because the backend was configured to only allow requests from the gateway IP.

**Fix:** Updated both config files:

- Frontend `.env`: changed API URL to `http://<GATEWAY_IP>/api`
- Backend `.env`: changed `CORS_ORIGIN` to `http://<GATEWAY_IP>`

Rebuilt both apps after the change. Once both ends were pointing to the gateway IP, everything worked end to end.

---

## Verification

After all fixes were applied, the following all worked correctly:

- `http://<GATEWAY_IP>/api/products` — returns JSON list of products from the database
- `http://<GATEWAY_IP>` — loads the ecommerce homepage
- `http://<GATEWAY_IP>/products` — products page loads and displays all items
- Signup form — submitting creates a new user, frontend calls `/api/auth/register` through the gateway, backend stores the user in Azure SQL, returns success

---

## What I Learned

**Networking is the hardest part of cloud deployments.** Most of the time I spent on this lab was not writing code or configuring the app — it was debugging why two services could not talk to each other. NSG rules, firewall rules, VNet routing, health probes — all of these layers exist to control what can talk to what, and getting them wrong causes silent failures that are hard to diagnose.

**Read the logs before guessing.** Every single issue in this lab was diagnosed by reading `pm2 logs backend`. The error messages were specific and pointed directly to the cause. The instinct to click around in the portal trying things randomly does not work — the answer is always in the logs.

**Azure SQL firewall rules only accept public IPs.** This is not obvious and not well documented. If you want to restrict database access to a specific VM without using its public IP (which changes if the VM is restarted), you need to use a Virtual Network rule instead. This is actually the better approach anyway because it ties access to the network topology rather than a specific IP address.

**Health probes matter.** The Application Gateway will not send traffic to a backend unless the health probe passes. If your app returns 404 on the probe path, the gateway marks the backend as unhealthy and returns 502 to users. Always configure a dedicated health check endpoint on your API and point the gateway probe at it.

**PM2 startup directory.** PM2 does not always correctly store the working directory when you register a startup command. Always `cd` into the application folder before running `pm2 start`. Verify the process is running from the right path with `pm2 info <name>`.

**CORS is a browser enforcement, not a server enforcement.** The backend was running fine and serving correct responses, but the browser refused to use those responses because the `Origin` header did not match what the backend allowed. Changing the CORS_ORIGIN on the backend and rebuilding fixed it immediately. Understanding that CORS is enforced client-side helps with debugging — you can always test API endpoints directly with `curl` even when the browser is blocking them.

**Subnets must be planned before you need them.** The Application Gateway requires its own dedicated subnet that cannot contain any other resources. I had to create `subnet-appgw` specifically for it. In a real deployment this subnet planning happens before anything is provisioned. Learning to think about subnet layout upfront is an important infrastructure habit.

**Private access to databases is the right default.** Putting the database behind a VNet rule and blocking all public access means that even if someone discovers the database hostname, they cannot connect to it at all from the internet. The backend is the only entry point. This is standard practice and something I now understand practically rather than just theoretically.

---

## Tools Used

- Azure Portal — resource provisioning and configuration
- Fedora Linux terminal — SSH, curl, nano
- PM2 — process manager for keeping Node.js apps running in the background
- Node.js / npm — runtime and package management
- git — cloning the application repository

---

*Lab completed as part of Ironhack DevOps Bootcamp, March 2026.*
