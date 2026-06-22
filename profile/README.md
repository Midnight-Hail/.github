# Midnight Hail

A defensive cyber suite for detection engineering, threat intelligence, and purple team operations — built around Splunk, SigmaHQ, and adversary emulation.

---

## Repositories

### Core Platform

| Repo | Description |
|------|-------------|
| [operations-suite](https://github.com/midnight-hail/operations-suite) | Docker-hosted mission console — the central hub. React SPA backed by Express that integrates all tools below into a single UI: Sigma→SPL converter, build pipeline, ATT&CK coverage heatmap, threat intel graph, rule builder, validator, purple team exercise management, and C2 auto-correlation via AdaptixC2. |

### Detection Engineering

| Repo | Description |
|------|-------------|
| [sigma-core](https://github.com/midnight-hail/sigma-core) | Shared Sigma→SPL translation library used by both `ta-sigma` and `ta-cti`. Contains all CIM field mappings, classification logic, and SPL generation code so fixes only need to be made once. |
| [ta-sigma](https://github.com/midnight-hail/ta-sigma) | Build tool that converts [SigmaHQ](https://github.com/SigmaHQ/sigma) detection rules into a deployable Splunk Technology Add-On (`Splunk_TA_SIGMA`). Classifies each rule into a CIM data model group and generates `savedsearches.conf` with `tstats summariesonly=t` searches. |
| [ta-cti](https://github.com/midnight-hail/ta-cti) | Build tool that converts Recorded Future threat intelligence lookup CSVs into a deployable Splunk TA (`Splunk_TA_CTI`). Clusters IOCs by actor and malware family, generates Sigma YAML, and produces tstats-based detection searches with lookup-join support for large IOC lists. |
| [detections-validator](https://github.com/midnight-hail/detections-validator) | Pre-deployment validator for `savedsearches.conf` files produced by `ta-sigma` and `ta-cti`. Catches SPL syntax errors, CIM field mapping mistakes, and tstats searches that would silently return zero results — before anything reaches the search head. |

### Splunk Add-Ons

| Repo | Description |
|------|-------------|
| [sa-powershell](https://github.com/midnight-hail/sa-powershell) | Splunk TA (`Splunk_TA_PowerShell`) that provides a custom `Powershell` CIM data model for accelerating PowerShell event log detections. Also ships the `\| psreconstruct` generating search command, which rebuilds full PowerShell scripts from fragmented 4104 Script Block Logging events. |
| [sa-ultimate-network-viz](https://github.com/midnight-hail/sa-ultimate-network-viz) | Splunk add-on providing network topology visualization dashboards. Vendored as a submodule in `operations-suite`. |

### Range Provisioning

| Repo | Description |
|------|-------------|
| [win-range](https://github.com/midnight-hail/win-range) | VMware vSphere range provisioner (Range Forge). Builds a full Windows Active Directory range using Packer, Terraform, and Ansible — with GOAD, GHOSTS, BadBlood, and Sysmon pre-configured. Orchestrated from the `operations-suite` Range Forge tab. |
| [k8s-range](https://github.com/midnight-hail/k8s-range) | Kubernetes-native vulnerable service provisioner (cyber-range). Deploys CVE-profiled web application containers (Juice Shop, Beelzebub, vulnerable Flask apps, Gitea misconfigs) into isolated K8s namespaces. Complements `win-range` to form a full kill-chain range: T1190 web exploit → lateral movement into the AD range. |

---

## How the Repos Fit Together

```
operations-suite  ← the mission console (Docker)
  ├── sigma-core          shared Sigma→SPL library
  ├── ta-sigma            SigmaHQ → Splunk_TA_SIGMA
  ├── ta-cti              Recorded Future → Splunk_TA_CTI
  ├── detections-validator  pre-deployment validation
  ├── sa-powershell       PowerShell data model + psreconstruct
  ├── sa-ultimate-network-viz  network viz dashboards
  ├── win-range           AD range (vSphere / GOAD)
  └── k8s-range           web target range (Kubernetes)
```

`operations-suite` vendors all sibling repos as git submodules under `dependencies/`. A single clone with `git submodule update --init --recursive` brings everything in.

---

## Quick Start

### Docker (full feature set)

```bash
git clone https://github.com/midnight-hail/operations-suite.git
cd operations-suite
git submodule update --init --recursive
docker compose build && docker compose up
```

Open `http://localhost:7860`. On first startup an `admin` account is created with a temporary password printed to stdout.

### Local Development (Node only)

Python-backed routes (builds, validator, coverage, CTI graph) are unavailable outside Docker. All other features work locally.

```bash
cd operations-suite
npm install
npm run install:all

# create server/.env
echo "DATA_DIR=./data`nPORT=8080" > server/.env

npm run dev
```

Open `http://localhost:5173`. Vite proxies all `/api` requests to the Express server at `8080`.

### Management Mode (license authority)

Set `MANAGEMENT_MODE=true` to enable the License Manager tab and `/api/license-mgmt/*` routes. This mode is intended for the issuing-authority instance only.

**PowerShell:**
```powershell
$env:MANAGEMENT_MODE = "true"
node server/dist/index.js
```

**Docker:**
```bash
MANAGEMENT_MODE=true docker compose up
```

---

## Python Tool Dependencies

Each Python tool can be installed and run independently.

### sigma-core (shared library — install first)

```bash
pip install -e ./sigma-core
# with Splunk deployment support:
pip install -e "./sigma-core[deploy]"
```

### ta-sigma

```bash
pip install -e ./sigma-core
pip install -e ./ta-sigma
python -m ta_sigma_builder --index windows --output-dir dist
```

### ta-cti

```bash
pip install -e ./sigma-core
pip install -e ./ta-cti
python convert_rf_to_sigma.py --lookups-dir ./lookups --output-dir rf_sigma_rules
python -m ta_cti_builder --sigma-root rf_sigma_rules --index windows
```

### detections-validator

```bash
pip install requests pyyaml
python validator.py --static-only --apps ../ta-sigma/dist/Splunk_TA_SIGMA
```

---

See the [operations-suite README](https://github.com/midnight-hail/operations-suite) for full setup and architecture details.
