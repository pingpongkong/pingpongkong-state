
# 🦍 PingPongKong State Repository

Welcome to the **Source of Truth** for the PingPongKong ecosystem. 

This repository contains the declarative **Connectivity Matrices** for our entire infrastructure. It defines exactly "who must be able to talk to whom" across both our Kubernetes clusters and legacy bare-metal environments. If a network path defined in this repository goes down, PingPongKong will detect the drift and alert us.

## 🏗️ The PingPongKong Ecosystem

This state data is consumed by the four core components of the PingPongKong suite:

| Component | Environment | Role |
| :--- | :--- | :--- |
| **`pingpongkong-k8s-agent`** | Kubernetes | Rust DaemonSet. Reads `k8s/*.yaml`, auto-discovers target IPs via K8s RBAC, and performs the actual TCP/UDP/ICMP probing. |
| **`pingpongkong-k8s-collector`** | Kubernetes | Aggregates data from K8s Agents. Compares actual state vs. the desired state in this repo. Updates K8s CRDs & sends Discord alerts. |
| **`pingpongkong-node-agent`** | Legacy/Bare-Metal | Lightweight static binary for VMs/Physical servers. Reads `node/*.yaml` and probes explicit, hardcoded IPs. |
| **`pingpongkong-node-collector`** | Legacy/Bare-Metal | Centralized aggregator for non-K8s environments. Collects metrics from Node Agents for unified dashboarding. |


## 🚀 How to Use in Production

Because this repository contains internal IP addresses, hostnames, and architecture maps, it should be kept secure. Follow these steps to deploy the PingPongKong ecosystem:

### 1. Clone to a Private Repository
Do not fork this publicly. Clone this repository and push it to a **Private Repository** within your organization's GitHub/GitLab workspace.

### 2. Modify the State Files
Edit the files in the `k8s/` and `node/` directories to match your actual infrastructure requirements. 
* Delete the sample files.
* Create your own cluster and node-group matrices.
* Ensure there are no validation errors in your editor.

### 3. Generate a Read-Only Token
The PingPongKong Agents and Collectors need access to read these YAML files securely.
* **GitHub:** Create a Fine-grained Personal Access Token (PAT) with `Contents: Read-only` access to your private state repository.
* **GitLab/Bitbucket:** Create a Repository Deploy Token with `read_repository` scope.

### 4. Deploy the Agents & Collectors
Pass the state repository URL and the token to your PingPongKong binaries. 

**For Kubernetes (via Secret):**
```yaml
env:
  - name: STATE_REPO_URL
    value: "[https://raw.githubusercontent.com/your-org/pingpongkong-state/main/k8s/my-cluster.yaml](https://raw.githubusercontent.com/your-org/pingpongkong-state/main/k8s/my-cluster.yaml)"
  - name: GITHUB_TOKEN
    valueFrom:
      secretKeyRef:
        name: pingpongkong-auth
        key: token
For Bare-Metal Nodes (via CLI or systemd):
```

Bash
```
./pingpongkong-node-agent \
  --config-url "[https://raw.githubusercontent.com/your-org/pingpongkong-state/main/node/legacy-db.yaml](https://raw.githubusercontent.com/your-org/pingpongkong-state/main/node/legacy-db.yaml)" \
  --auth-token "github_pat_XXXXXXX"
```

5. Drift Detection & Alerting
Once deployed, the Agents will continuously fetch the YAML state using the token. They will perform high-speed TCP/UDP/ICMP handshakes against the defined matrix. If an actual network path fails (Network Drift) or a YAML configuration changes (State Drift), the Collectors will instantly aggregate the failure and trigger your Discord webhooks or PagerDuty alerts.

---

## 📂 Repository Structure

We maintain a flat, easily searchable file structure separated by environment type:

```text
pingpongkong-state/
├── k8s/                        # Targets discovered dynamically via K8s API
│   ├── sample-k8s-cluster.yaml
│   └── sample-prod-cluster.yaml
├── node/                       # Explicit, static targets for Bare-Metal
│   ├── sample-legacy-db.yaml
│   └── sample-baremetal-storage.yaml
├── schemas/                    # JSON Schemas for strict YAML validation
│   ├── k8s-schema.json
│   └── node-schema.json
├── notification/               # NEW: Alert routing and limits
│   └── discord.yaml
└── .vscode/                    # Editor settings for automatic linting
    └── settings.json
```

---

## ✍️ How to Add or Modify Rules

### 1. Kubernetes Environments (`k8s/`)
Kubernetes matrices use **Role Labels** (`node-role.kubernetes.io/xyz`) for dynamic IP discovery. The agent automatically figures out the IPs based on the cluster state.

**Example `k8s/sample-k8s-cluster.yaml`:**
```yaml
version: "1.0"
cluster: "sample-k8s-cluster"
topology:
  roles:
    controlplane: "node-role.kubernetes.io/control-plane"
    worker: "node-role.kubernetes.io/worker"
matrix:
  internal:
    - description: "Kubelet API for Logs/Exec"
      from: "controlplane"
      to: "worker"
      ports: [10250]
      proto: "tcp"
```

### 2. Bare-Metal & Legacy Environments (`node/`)
Since these environments lack an API for dynamic discovery, targets are explicitly listed in arrays.

**Example `node/sample-group01.yaml`:**
```yaml
version: "1.0"
node_group: "sample-group01"
matrix:
  - description: "MySQL Primary to Replica Sync"
    from: 
      - "10.50.0.10"
      - "10.50.0.11"
    to: 
      - "10.50.0.12"
      - "10.50.0.13"
    ports: [3306]
    protocol: "tcp"
```

---

## 🛡️ Strict Validation (Action Required for Developers)

To prevent typos (like writing `protocol: tpc`) from crashing the Rust agents in production, this repository enforces strict JSON Schema validation. **No unknown keys or invalid protocols are allowed.**

### VS Code Setup
If you use VS Code, validation is built-in. Simply ensure you have the **Red Hat YAML Extension** installed (`redhat.vscode-yaml`). 

The `.vscode/settings.json` file in this repository will automatically map:
* Files in `k8s/` to `schemas/k8s-schema.json`
* Files in `node/` to `schemas/node-schema.json`

If you make a formatting error, VS Code will immediately highlight the line in red. **Do not commit YAML files that contain validation errors.**

---

## 🤝 Contributing
1. Create a new branch for your matrix updates.
2. Add or modify the `.yaml` files in the appropriate directory.
3. Ensure your editor shows no schema validation errors.
4. Open a Pull Request for SRE review.
