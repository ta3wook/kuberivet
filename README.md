# Kuberivet

Simple Ansible automation for bootstrapping Kubernetes clusters in homelabs.

Kuberivet bootstraps existing Debian VMs into a Kubernetes cluster with one
control-plane node, worker nodes, containerd, and Calico networking.
Each stage can also be run as a separate playbook.

The initial homelab workflow is implemented and has completed successfully
in the development environment, including cross-node Pod traffic, Service
access, and DNS resolution. HA and lifecycle operations remain outside the
current scope.

## Current scope

| Stage | What it does |
| --- | --- |
| Node preparation | Disable swap and configure kernel modules and sysctl settings |
| Container runtime | Install containerd and configure the systemd cgroup driver |
| Kubernetes packages | Install and hold kubeadm and kubelet, with kubectl on control-plane nodes |
| Control plane | Deploy kubeadm configuration, run initialization, and configure a dedicated kubectl administrator |
| Calico | Bootstrap CRDs and Tigera Operator, apply network settings, and wait for availability |
| Workers | Generate a short-lived join command when needed, join workers, and wait for Ready |

## Requirements

**Controller** (the machine running Ansible):

- A Linux environment with Git, OpenSSH, Python, and Python venv support.
- The current development environment uses `ansible-core 2.19.11`.
  Its supported controller Python versions are 3.11–3.13;
  see the [Ansible support matrix](https://docs.ansible.com/projects/ansible/latest/reference_appendices/release_and_maintenance.html).
- SSH access to every target VM, and internet access for dependency downloads.

**Target VMs:**

- Current package settings target **Debian 13 (Trixie)**. Other distributions
  have not been validated.
- Python 3 at `/usr/bin/python3`, systemd, and access to the Debian, Docker,
  and Kubernetes package repositories.
- An existing SSH account with root privileges or sudo access.
  The inventory currently uses `ansible`; change it to match your environment.

The sample inventory contains one control-plane host and two workers.
Use exactly one control-plane host for the current workflow; HA control-plane
joining is not implemented. Worker node names are set from inventory hostnames,
so use valid Kubernetes node names.
VM creation and provisioning the Ansible SSH account are outside the playbooks.

## Quick start

Run the following commands on the controller. Run Ansible commands from the
repository root so that `ansible.cfg` resolves the inventory and roles correctly.

### 1. Clone and install dependencies

```bash
git clone https://github.com/ta3wook/kuberivet.git
cd kuberivet
python3 -m venv "$HOME/.venvs/kuberivet"
. "$HOME/.venvs/kuberivet/bin/activate"
python -m pip install 'ansible-core==2.19.11'
ansible-galaxy collection install -r requirements.yml
```

If Ansible is already installed, check `ansible --version` and install the
collections from `requirements.yml` in that environment.

### 2. Configure your inventory

```bash
cp examples/hosts.yml.example inventory/hosts.yml
```

Edit `inventory/hosts.yml` to match your VMs. The example contains one
control-plane host and two workers, using documentation-only IP addresses.
Replace every `ansible_host` with the corresponding VM's actual address.
Add or remove entries under `control_plane.hosts` and `workers.hosts` to
match your worker count; keep one control-plane host and both groups under
the parent `k8s` group.

If needed, add `ansible_user`, `ansible_port`, or
`ansible_ssh_private_key_file` to a host entry. Verify each server's SSH host
key before accepting it on the first connection.

Review `inventory/group_vars/k8s.yml` for the common account and package versions.
Set `kubernetes_admin_user` in `inventory/group_vars/control_plane.yml` to the
non-root account that should use kubectl on the control-plane VM. The role
creates this account and places the admin kubeconfig at
`/home/<kubernetes_admin_user>/.kube/config`. It does not configure SSH access
for that account. This kubeconfig grants cluster-admin access; keep it private.
The containerd version includes a Debian release suffix.
For Kubernetes, set `kubernetes_version` to the upstream version without the
`v` prefix (for example, `1.37.1`) and `kubernetes_package_revision` to the APT
package revision (for example, `1.1`). The package role combines them as
`1.37.1-1.1`; the kubeadm template uses `v1.37.1` without the package revision.
The Kubernetes repository minor version is currently fixed in
`roles/kubernetes/tasks/main.yml`; changing `kubernetes_version` alone does
not change that repository. Check that your selected package versions are
available before provisioning.

`kubernetes_pod_cidr` is shared by kubeadm and Calico. Choose a range that does
not overlap your VM or Service networks, and keep the existing range when
rerunning against an initialized cluster. `calico_version` selects the Calico
manifests. The Calico role installs `acl` and creates the dedicated Python
client environment configured by `kubernetes_client_venv` and
`kubernetes_client_python` on the control-plane VM.

The actual `inventory/hosts.yml` is ignored by Git, including its node list,
addresses, and group membership. Preserve a separate private backup.
Only `examples/hosts.yml.example` is published as a reusable template.
Keep private keys outside the repository and avoid plaintext passwords.

Most nodes do not need a `host_vars` file. When a host needs additional public
settings, create `inventory/host_vars/<hostname>/main.yml`, using exactly the
same hostname as in the inventory. Common settings belong in `group_vars`.
Keep connection details in the actual inventory rather than duplicating
variables in host files. The old `host_vars/*/local.yml` path remains ignored
as a safeguard, but is no longer used by this setup.

### 3. Check the inventory and connection

```bash
ansible-playbook playbooks/site.yml --list-hosts
ansible k8s -m ansible.builtin.ping
ansible-playbook --syntax-check playbooks/site.yml
```

Confirm that the host list matches your intended VMs and every host returns
`pong`. The ping module checks SSH and Python availability, not just ICMP.
Syntax checking does not verify package availability or prove installation will succeed.

### 4. Bootstrap the cluster

These playbooks change target VM settings, may reboot nodes during swap
configuration, initialize the control plane, install Calico, and join workers.
Use VMs you intend to configure for this homelab.

```bash
ansible-playbook playbooks/site.yml
```

Add `--ask-become-pass` if sudo requires a password. Do not store the password
in public inventory files.

To run one stage independently:

```bash
ansible-playbook playbooks/01-node-prep.yml
ansible-playbook playbooks/02-install-containerd.yml
ansible-playbook playbooks/03-install-kubernetes.yml
ansible-playbook playbooks/04-init-control-plane.yml
ansible-playbook playbooks/05-install-calico.yml
ansible-playbook playbooks/06-join-workers.yml
```

On fresh VMs, run the stages in the order shown. Initialization is skipped
when `/etc/kubernetes/admin.conf` already exists. This guard does not check
cluster health or recover a partially initialized control plane.
Worker joining is skipped when `/etc/kubernetes/kubelet.conf` exists. Join
commands expire after 30 minutes and are hidden from normal Ansible logs.
Calico bootstrap manifests are skipped when the installation CRD and Operator
Deployment respectively exist; custom resources and readiness checks still run.
These guards do not implement cluster recovery or Calico upgrades.

To run worker joining for a subset, include the control plane in the limit
because it generates the join command and checks node readiness:

```bash
ansible-playbook playbooks/06-join-workers.yml \
  --limit 'k8s-cp01,k8s-worker01' -K
```

## Verify the result

After the play recap shows no failed or unreachable hosts, open a session as
the configured Kubernetes administrator on the control-plane VM and run:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

The control plane and every configured worker should be Ready. System Pods,
including Calico and CoreDNS, should have their containers ready. Worker nodes
may display `<none>` in the ROLES column until a role label is added.
A successful node check does not by itself prove Pod traffic or DNS works;
validate those with test workloads before relying on the cluster.

## Troubleshooting

- **SSH or permission errors:** check the host entry in `inventory/hosts.yml`,
  SSH key, account, and sudo access. Use `--ask-become-pass` when required.
- **Missing module:** run `ansible-galaxy collection install -r requirements.yml`
  using the same Ansible environment.
- **Package version not found:** check the configured version, Debian release,
  and repository minor version. Use the upstream version for kubeadm and the
  full version including `kubernetes_package_revision` for APT packages.
- **Node is NotReady:** check the failed Calico or worker readiness task,
  then inspect `kubectl get pods -A` on the control plane. A node can be
  temporarily NotReady while networking starts.
- **Worker subset execution:** include the control-plane host in `--limit`;
  it is needed even when only one worker is being joined.
- **Template deprecation warning:** the tested `kubernetes.core 6.6.0` template
  path emits a `Templar.environment` warning with ansible-core 2.19.11.
  The validation run completed despite this warning; track the
  [upstream issue](https://github.com/ansible-collections/kubernetes.core/issues/1208)
  before changing the core or collection version.

## Validation status

The following results were confirmed manually in the development homelab on
2026-11-27:

| Component | Validated environment |
| --- | --- |
| Topology | One control-plane VM and two workers |
| Target OS | Debian GNU/Linux 13 (Trixie) |
| Kubernetes | v1.37.1 |
| Container runtime | containerd 2.3.6 |
| Calico | v3.33.1 |

- The full playbook reached its final recap with no failed or unreachable hosts.
- All three nodes reported Ready and all listed system Pods were Running and ready.
- A client Pod on one worker reached a web Pod on the other worker with HTTP 200.
- Cluster DNS resolved the web Service, and access through the Service returned HTTP 200.

Inventory loading, template rendering, playbook syntax, bootstrap guards,
readiness conditions, and worker selection have also been checked without
applying changes to the cluster.

A fresh-VM walkthrough of this README and a full rerun after all workers joined
have not yet been verified. Validation does not cover other distributions,
HA, upgrades, reset/recovery, storage workloads, or production use.
