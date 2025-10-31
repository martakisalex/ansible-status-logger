# Ansible Logger and Insights

A lightweight, agentless logging and analysis pipeline built in Ansible.  
It collects Kubernetes and Helm command outputs, consolidates them into a master JSON index, analyzes those snapshots for insights and diffs, and generates both human-readable and machine-readable reports.

---

## Overview

This project runs locally or on a VM (macOS, RHEL, or any Linux system) and provides:
- Structured JSON data for Helm and Kubernetes states (`index.json`)
- Computed diffs between snapshots (added/removed/changed pods and releases)
- Summarized Markdown and HTML reports (`logs/reports/`)
- Machine-readable alert summaries (`alerts.json`)

The pipeline is divided into three Ansible roles:

| Role | Purpose |
|------|----------|
| **collector_commands** | Executes `helm` and `kubectl` commands to gather logs from the cluster and store them under `logs/`. |
| **analyzer_files** | Merges and normalizes collected logs into a unified master file (`logs/index.json`) for structured analysis. |
| **analyzer_insights** | Reads the master JSON, calculates diffs and unhealthy components, and generates `alerts.json`, Markdown, and HTML reports. |

---

## Project Structure

```
ansible-logger/
├── inventory.ini
├── site.yml
├── group_vars/
│   └── all.yml
├── roles/
│   ├── collector_commands/
│   ├── analyzer_files/
│   └── analyzer_insights/
└── logs/
    ├── helm/
    ├── kubernetes/
    ├── reports/
    └── index.json
```

---

## Dependencies

### Required Packages

Ensure these dependencies are installed **on the control host (your local machine or VM)**:

| Package | Purpose | Install Command (RHEL) | Install Command (macOS) |
|----------|----------|------------------------|--------------------------|
| `ansible-core` | Runs playbooks | `pip3 install --user ansible-core` | `pip3 install --user ansible-core` |
| `jq` | JSON processing | `sudo dnf install jq` or `sudo dnf install epel-release jq` | `brew install jq` |
| `bash`, `awk`, `sed`, `grep` | Standard GNU tools | Included on RHEL | Included on macOS |

If you plan to collect live cluster data, also install:
```bash
sudo dnf install kubectl helm
```

### Python
Ensure Python 3.6+ is installed and in your PATH:
```bash
python3 --version
```

---

## Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/<your-username>/ansible-logger.git
   cd ansible-logger
   ```

2. **Set configuration values**
   Edit `group_vars/all.yml` to define paths:
   ```yaml
   log_dir: "{{ playbook_dir }}/logs"
   analysis_output_json: "{{ log_dir }}/index.json"
   report_dir: "{{ log_dir }}/reports"
   insights_min_ready_ratio: 1.0
   insights_alert_on_status: ["CrashLoopBackOff", "Error", "Terminating"]
   insights_helm_bad_status: ["failed", "pending-upgrade", "pending-rollback"]
   ```

3. **Create directories**
   ```bash
   mkdir -p logs/{helm,kubernetes,reports}
   ```

4. **(Optional) Verify jq installation**
   ```bash
   jq --version
   ```

---

## Running the Pipeline

### Run Everything
To execute the full pipeline (collection → file analysis → insights):
```bash
ansible-playbook -i inventory.ini site.yml
```

This runs all roles in order:
1. `collector_commands`
2. `analyzer_files`
3. `analyzer_insights`

---

## Running Individual Roles

Roles can be executed independently using tags.

### site.yml Example with Tags
```yaml
- hosts: localhost
  gather_facts: false
  roles:
    - { role: collector_commands, tags: ['collector'] }
    - { role: analyzer_files, tags: ['files'] }
    - { role: analyzer_insights, tags: ['insights'] }
```

### Run a Single Role
- Collector only:
  ```bash
  ansible-playbook -i inventory.ini site.yml --tags collector
  ```
- Analyzer (file merge only):
  ```bash
  ansible-playbook -i inventory.ini site.yml --tags files
  ```
- Insights and reporting only:
  ```bash
  ansible-playbook -i inventory.ini site.yml --tags insights
  ```

### Skip Roles
To run everything **except** a role:
```bash
ansible-playbook -i inventory.ini site.yml --skip-tags collector
```

---

## Outputs

After a full run, the following files are generated:

| File | Description |
|------|--------------|
| `logs/index.json` | Master combined JSON containing all Helm/Kubernetes data. |
| `logs/reports/report-<UTC>.md` | Human-readable Markdown report with current insights. |
| `logs/reports/latest.md` | Convenience copy of the most recent report. |
| `logs/reports/report-<UTC>.html` | HTML version of the report with color-coded statuses (green/yellow/red). |
| `logs/reports/alerts.json` | Machine-readable summary for integration with CI/CD or ChatOps. |

---

## Example Use Cases

- **Pre-Deployment Checks:** Run before and after a Helm upgrade to confirm pod health and release integrity.
- **CI/CD Pipelines:** Integrate with Jenkins, GitHub Actions, or GitLab to automatically fail builds on unhealthy pods or failed Helm releases.
- **Incident Reports:** The Markdown/HTML outputs provide clear before/after comparisons and health summaries.

---

## Compatibility

| Environment | Status |
|--------------|--------|
| macOS (Homebrew) | ✅ Fully supported |
| RHEL / CentOS / Fedora | ✅ Fully supported |
| Ubuntu / Debian | ✅ Works with minor package name differences |
| Kubernetes / Helm versions | ✅ Tested with Helm 3 and K8s 1.27+ |

---

## Extending the System

To extend functionality:
- Add new resource collectors (e.g., `kubectl get deployments`, `kubectl get ingress`).
- Update `analyzer_files` to parse additional data structures.
- Add new computed insights in `analyzer_insights/tasks/main.yml`.
- Create templates under `roles/analyzer_insights/templates/` for custom report formats.

---

## License

MIT License.  
Developed by Alex Martakis.

---

## Notes

- This project is fully agentless — no daemons or in-cluster components required.
- Designed for local runs, CI/CD stages, or secure jump hosts.
- Compatible with both offline and online cluster analysis workflows.
