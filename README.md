# F5 AS3 Lab Guide: Standardizing LTM & APM Configurations via GitOps

## Executive Summary & Context

Modern application deployment requires speed, consistency, and security governance across isolated network enclaves. Traditional imperative configuration methods (GUI point-and-click or step-by-step TMSH commands) often result in configuration drift, human error, and slow release cycles.

This lab guide introduces **F5 Application Services 3 Extension (AS3)**—a declarative configuration management tool—integrated into a GitOps workflow using your on-site GitLab instance (`10.1.20.150`). By storing declarative JSON definitions in Git, product development teams gain visibility and self-service capabilities via automated CI/CD pipelines, while network and security teams maintain strict governance over shared BIG-IP LTM and APM infrastructure across security enclaves.

---

## Lab Environment Overview

The lab environment consists of an F5 BIG-IP system, a client system, and a Docker host hosting backend web applications and CI/CD tools.

### Network Topology & Infrastructure

- **BIG-IP Management IP:** `10.1.1.245`
- **BIG-IP Self IPs:**
  - **External:** `10.1.10.31/24` (Self), `10.1.10.33/24` (Floating)
  - **Internal:** `10.1.20.31/24` (Self), `10.1.20.33/24` (Floating)
- **Client System (`ubuntu-client`):** `10.1.10.15` (External), `10.1.20.15` (Internal)
- **On-Premise GitLab Server:** `http://10.1.20.150` (SSH: port `2424`)
- **GitLab Runners:** `gitlab-runner-1`, `gitlab-runner-2`, `gitlab-runner-3`
- **Backend NGINX Web Servers (Docker host `10.1.20.25`):**
  - `server1` - `server12`: IPs `10.1.20.101` through `10.1.20.112` on port `80`

```
+---------------------------------------------------------------------------------+
|                                 LAB TOPOLOGY                                    |
|                                                                                 |
|  [ Ubuntu Client ] <---> [ BIG-IP (10.1.10.33 / 10.1.20.33) ]                   |
|  (10.1.10.15)                   |                                               |
|                                 | (Internal VLAN: 10.1.20.0/24)                 |
|                                 +-----------------------+                       |
|                                 |                       |                       |
|                       [ GitLab Server ]       [ Docker Web Cluster ]            |
|                         (10.1.20.150)           (10.1.20.101 - 112)             |
+---------------------------------------------------------------------------------+
```

---

## Module 1: F5 AS3 Concepts & Architecture

### What is F5 AS3?

F5 Application Services 3 Extension (AS3) is a flexible, low-overhead mechanism for deploying application-specific configurations on BIG-IP systems using a declarative API.

#### Declarative vs. Imperative

- **Imperative:** You tell the system *how* to build a configuration step-by-step (e.g., "Create node X", "Create pool Y with node X", "Create virtual server Z using pool Y").
- **Declarative:** You define the *desired end state* in JSON format (e.g., "I need an HTTP virtual server at IP 10.1.20.200 pointing to pool members 10.1.20.101 and 10.1.20.102"). AS3 automatically calculates the delta and applies or updates necessary objects.

### How AS3 Operates

1. **REST Request:** A client or CI/CD pipeline sends a `POST` or `DECLARATION` API call containing JSON to `https://<BIG-IP-IP>/mgmt/shared/appsvcs/declare`.
2. **Validation:** The AS3 engine validates the payload against strict JSON Schema definitions.
3. **Execution:** AS3 converts the JSON declaration into TMSH commands internally, modifies BIG-IP memory and disk configuration, and creates an isolated partition/tenant.

### AS3 Hierarchy Structure

```
ADC (Application Delivery Controller)
└── Tenant (Partition / Security Enclave)
    └── Application
        ├── Service (Virtual Server)
        ├── Pool
        ├── Monitor
        └── TLS / APM Policies
```

---

## Module 2: Setting Up the GitLab GitOps Repository

In this module, you will configure the on-site GitLab instance (`10.1.20.150`) to host your AS3 declarations and automate their deployment to the BIG-IP system using GitLab CI/CD pipelines.

### Step 2.1: Create the GitLab Project

1. Open a browser and navigate to `http://10.1.20.150`.
2. Log in with your developer or administrator credentials.
3. Click **New Project** > **Create Blank Project**.
4. Project name: `f5-as3-declarations`.
5. Visibility Level: **Internal** or **Private**.
6. Click **Create project**.

### Step 2.2: Configure CI/CD Environment Variables

To keep credentials secure and prevent hardcoding secrets in Git repositories:

1. In your GitLab project, go to **Settings** > **CI/CD**.
2. Expand the **Variables** section and add the following variables:
   - `BIGIP_MGMT_IP`: `10.1.1.245`
   - `BIGIP_USER`: `admin`
   - `BIGIP_PASS`: `<your-bigip-password>` (Mark as **Masked**)

---

## Module 3: AS3 Declarations & GitOps Pipeline Setup

### Exercise 3.1: Basic HTTP Application Load Balancing

File: [`declaration_1_http.json`](declaration_1_http.json)

Creates tenant `Enclave_Dev`, application `HTTP_Service`, Virtual Server on `10.1.20.200:80`, and pool members `10.1.20.101` and `10.1.20.102`.

### Exercise 3.2: HTTPS Application with TLS Termination

File: [`declaration_2_https.json`](declaration_2_https.json)

Deploys a secure HTTPS Virtual Server on `10.1.20.201:443`, imports TLS cert/key dynamically, and configures pool members `10.1.20.103` and `10.1.20.104`.

### Exercise 3.3: Multi-Tenant Enclave Isolation & APM Security Integration

File: [`declaration_3_multi_enclave_apm.json`](declaration_3_multi_enclave_apm.json)

Isolates resources across `Enclave_Dev`, `Enclave_Prod`, and `Enclave_Secure`. Binds `/Common/Global_APM_Policy` access policy to the secure HTTPS Virtual Server (`10.1.20.210:443`).

---

## Module 4: Configuring the GitLab CI/CD Pipeline

The repository includes a [`.gitlab-ci.yml`](.gitlab-ci.yml) file that validates JSON syntax and automatically deploys declarations via `curl` to BIG-IP.

---

## Module 5: Git Workflows & Verification Steps

### Step 5.1: Clone, Commit, and Push

```bash
# Clone the repository from your on-site GitLab server
git clone http://10.1.20.150/root/f5-as3-declarations.git
cd f5-as3-declarations

# Add declaration files and pipeline configuration
git add declaration_1_http.json declaration_2_https.json declaration_3_multi_enclave_apm.json .gitlab-ci.yml

# Commit changes
git commit -m "feat: add initial AS3 declarations for Dev, Prod, and Secure enclaves"

# Push to GitLab (triggers CI/CD pipeline)
git push origin main
```

### Step 5.2: Monitor Pipeline Execution

1. Navigate to GitLab (`http://10.1.20.150`).
2. Open your project and click **Build** > **Pipelines**.
3. View job logs for `validate_declarations` and `deploy_as3_dev`.

---

## Module 6: BIG-IP Verification & Traffic Testing

### Step 6.1: Direct cURL Submission (Manual Alternative)

```bash
curl -k -u admin:${BIGIP_PASS} \
  -X POST \
  -H "Content-Type: application/json" \
  -d @declaration_1_http.json \
  https://10.1.1.245/mgmt/shared/appsvcs/declare
```

### Step 6.2: BIG-IP CLI Verification

```bash
ssh admin@10.1.1.245

# List created partitions / tenants
tmsh list sys folder

# Inspect Virtual Servers inside the Enclave_Dev partition
tmsh list ltm virtual /Enclave_Dev/HTTP_Service/serviceMain

# Inspect Pool Status and Backend Container Members
tmsh show ltm pool /Enclave_Dev/HTTP_Service/web_pool members
```

### Step 6.3: Client Traffic Validation

From `ubuntu-client` (`10.1.10.15`):

```bash
# Test HTTP Load Balancing Virtual Server
curl -I http://10.1.20.200

# Send multiple requests to verify load balancing across server1 (10.1.20.101) and server2 (10.1.20.102)
for i in {1..6}; do curl -s http://10.1.20.200 | grep -i "Server Name\|Host"; done
```

---

## Best Practices for Production GitOps

1. **Source Control Everything:** Store all AS3 JSON files in Git repositories for auditability and rapid rollbacks (`git revert`).
2. **Enforce Tenant Partitioning:** Map each security enclave to a distinct AS3 tenant for administrative isolation.
3. **Branch Protections & Merge Requests:** Require MR reviews prior to merging into `main`.
4. **Use Environment Secrets:** Mask credentials as GitLab CI/CD variables.
