# Kuberivet

Simple Ansible automation for bootstrapping Kubernetes clusters in homelabs.

Kuberivet prepares existing VMs for Kubernetes and initializes a single
control plane, with separate playbooks for each stage.

> **Work in progress:** node preparation and control-plane initialization are
> implemented. Worker joining and CNI installation are not yet automated.
> Running the playbooks does not yet produce a workload-ready cluster.

## Current scope

| Stage | What it does |
| --- | --- |
| Node preparation | Disable swap and configure kernel modules and sysctl settings |
| Container runtime | Install containerd and configure the systemd cgroup driver |
| Kubernetes packages | Install and hold kubeadm and kubelet, with kubectl on control-plane nodes |
| Control plane | Deploy kubeadm configuration, run initialization, and configure a dedicated kubectl administrator |

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
joining is not implemented. Workers receive node preparation and packages,
but are not yet joined to the cluster.
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

### 4. Prepare the nodes and initialize the control plane

These playbooks change target VM settings, may reboot nodes during swap
configuration, and initialize Kubernetes on the control-plane VM. Use VMs
you intend to configure for this homelab.

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
```

On fresh VMs, run the stages in the order shown. Initialization is skipped
when `/etc/kubernetes/admin.conf` already exists. This guard does not check
cluster health or recover a partially initialized control plane.

## Verify the result

```bash
ansible k8s -b -m ansible.builtin.command -a 'swapon --show'
ansible k8s -b -m ansible.builtin.command -a 'systemctl is-active containerd'
ansible k8s -m ansible.builtin.command -a 'dpkg-query -W containerd.io kubeadm kubelet'
```

Expected: no active swap entries, `active` for containerd, and installed
versions for the three packages on every node. The play recap should show
no failed or unreachable hosts. These checks cover node preparation only.
Check the initialized API server from the controller:

```bash
ansible control_plane -b -m ansible.builtin.command -a 'kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes'
```

The control-plane node should be listed. It can remain `NotReady`, and CoreDNS
can remain pending, until a CNI plugin is installed. Workers will not appear
until they are joined. Worker kubelets may restart while waiting for joining.
On the control-plane VM, the configured administrator can also run
`kubectl get nodes` using the kubeconfig in their home directory.

## Troubleshooting

- **SSH or permission errors:** check the host entry in `inventory/hosts.yml`,
  SSH key, account, and sudo access. Use `--ask-become-pass` when required.
- **Missing module:** run `ansible-galaxy collection install -r requirements.yml`
  using the same Ansible environment.
- **Package version not found:** check the configured version, Debian release,
  and repository minor version. Use the upstream version for kubeadm and the
  full version including `kubernetes_package_revision` for APT packages.
- **Node is NotReady after initialization:** CNI installation and worker joining
  are still manual follow-up steps; the current playbooks do not automate them.

## Validation status

Control-plane initialization has been run successfully in the development
homelab. Inventory loading, template rendering, and playbook syntax have also
been checked. A fresh-VM, end-to-end walkthrough of this guide has not yet
been independently verified. CNI installation and worker joining remain
outside the current automated workflow.
