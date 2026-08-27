RHEL Infrastructure Automation Lab

A production-style, multi-node RHEL environment fully provisioned and configured with Ansible — built to demonstrate infrastructure-as-code, idempotent configuration management, and air-gapped operational patterns.

Every layer of this environment is declarative, version-controlled, and rebuildable from code: point the automation at bare virtual machines, run one command, and the entire environment configures itself and converges to a known-good state.

What This Demonstrates
Infrastructure as Code — the complete environment is defined in version-controlled Ansible, not manual configuration
Idempotent configuration management — roles declare desired state and converge to it; re-runs make zero changes when the system already matches
Air-gapped operations — a self-hosted package mirror lets managed nodes operate with no internet access, mirroring restricted/classified environments
Secrets management — sensitive data is encrypted at rest with Ansible Vault and safe to commit to version control
Modular design — reusable roles orchestrated through a multi-play master playbook
Architecture
                    ┌─────────────────────────────────────────────┐
                    │  control  (192.168.56.10)                    │
                    │  ansible-navigator · execution environment   │
                    │  git · VS Code (remote) · project source     │
                    └───────────────────┬─────────────────────────┘
                                        │  SSH (key-based, sudo)
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
          ┌─────────▼────────┐ ┌────────▼────────┐ ┌────────▼─────────┐
          │ node1 (.11)      │ │ node2 (.12)     │ │ services (.13)   │
          │ webserver        │ │ dbserver        │ │ package mirror   │
          │ +5GB LVM disk    │ │ +5GB LVM disk   │ │ local DNF repo   │
          └──────────────────┘ └─────────────────┘ └──────────────────┘
                    │                   │                   ▲
                    └───────────────────┴───────────────────┘
                       managed nodes pull packages from the
                       local mirror — no internet required

4 nodes, built from a single golden image via full-clone:

control — Ansible control node running ansible-navigator with a containerized execution environment
node1 / node2 — managed nodes (webserver / dbserver), each with a dedicated disk for LVM storage automation
services — self-hosted infrastructure: a local RHEL package mirror served over HTTP
What's Automated

Each capability is a self-contained, reusable Ansible role:

Role	Capability
repo_client	Configures nodes to pull packages from the local mirror (air-gapped pattern)
base_config	Baseline packages, timezone, and system configuration applied to every node
user_mgmt	User and group management, driven by variables
storage_config	Full LVM stack — physical volume, volume group, logical volume, filesystem, and a reboot-persistent mount — with block/rescue error handling
web_server	Apache deployment with a Jinja2-templated config and a change-triggered restart handler
firewall_config	firewalld service and rule management, conditional on OS family
secure_info	Renders configuration from facts + Ansible Vault-encrypted secrets via Jinja2 templates
db_config	Database-server configuration combining facts, vault, and templating
motd_config	Standardized message-of-the-day deployment

All roles are orchestrated through site.yml — a multi-play playbook that targets the correct host groups and runs the full configuration in dependency order.

Key Engineering Decisions
<!-- JEAN: this section is the one that separates you from people who just followed a tutorial. I've drafted starting points below — rewrite these in YOUR words, from what you actually learned building this. Authenticity here matters more than polish. -->
Golden-image clone strategy. Rather than installing RHEL from ISO for each of the four nodes — a slow, repetitive process — I built one fully-configured base image and full-cloned it, applying the "if you do it more than once, automate it" principle from the outset. Cloning duplicates machine identity, so each clone required resetting anything that must be unique per host: hostname, machine-id, SSH host keys, network configuration, and subscription-manager identity. Beyond the time savings, this establishes reproducibility — a known-good, approved baseline that can be spun up consistently and identically on demand — and, combined with the version-controlled playbooks, means any node can be destroyed and rebuilt from code rather than treated as irreplaceable.
Local package mirror for air-gapped operation. Managed nodes are configured to pull all packages from a self-hosted mirror on the services node rather than the internet — deliberately reproducing the disconnected environments I operate professionally in DoD/classified work. Beyond enabling nodes to function without internet access, this enforces a controlled software supply chain: only vetted, approved packages are served to the environment, reducing attack surface and ensuring no unapproved software reaches sensitive systems. This mirrors the operational and security constraints of regulated environments, where controlling exactly what software enters production is a compliance requirement, not just a preference.
Idempotency as drift detection and compliance verification. Because every role declares desired state rather than executing imperative steps, running site.yml against an already-configured environment reports changed=0 — confirming the system still matches its approved baseline. This makes the automation a configuration-drift detector: any unauthorized or accidental deviation surfaces as a reported change on re-run, and because the roles declare state, the same run also remediates the drift back to the approved configuration. In a monitored or regulated environment, this doubles as a repeatable, on-demand compliance audit — you can prove a system's configuration hasn't deviated, or bring it back into compliance, with a single command.
Verification against the actual requirement, not the tool's report. Storage automation is validated by rebooting the node and confirming the mount returns — not merely by confirming it mounted during the run. A successful mount and a persistent mount are different claims with different failure modes: persistence depends on a correct /etc/fstab entry, which "it mounted" doesn't prove. Because a reported success can mask a human error — an unsaved change, a typo, a missing persistence step — I verify the real end state against the real requirement. This reflects a broader discipline applied throughout the project: "changed" is not "correct," and critical outcomes are confirmed by testing the condition they must actually satisfy.
How to Run It
bash
# Install dependencies (collections are not committed — they're declared)
ansible-galaxy collection install -r requirements.yml -p ./collections

# Verify connectivity to managed nodes
ansible-navigator run playbooks/ping.yml -m stdout

# Configure the entire environment
ansible-navigator run site.yml -m stdout --ask-vault-pass

Running site.yml a second time demonstrates idempotency: a fully-configured environment reports changed=0.

Skills Demonstrated

Configuration Management & IaC: Ansible (roles, playbooks, ansible-navigator, execution environments, Vault, Jinja2 templates, handlers, conditionals, error handling, facts), infrastructure-as-code, idempotent design

Linux Systems Administration: RHEL 9, LVM storage, systemd, firewalld, SELinux, package management, user/group administration

Tooling & Workflow: Git version control, VS Code Remote-SSH, self-hosted package mirroring, virtualization (VirtualBox)

Operational Patterns: air-gapped environments, secrets management, dependency management, multi-node orchestration

Certifications
Red Hat Certified System Administrator (RHCSA) — EX200
Red Hat Certified Engineer (RHCE) — EX294 (in progress)
CompTIA Security+