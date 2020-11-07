# Hetzner Bare Metal k8s Cluster

The scripts in this repository will setup and maintain one or more [kubernetes][k8s] clusters consisting of dedicated [Hetzner][hetzner] servers. Each cluster will also be provisioned to operate as a node in the [THORCHain][tc] network.

Executing the scripts in combination with some manual procedures will get you highly available, secure clusters with the following features on bare metal.

* [Kubespray][kubespray] (based)
* Internal NVMe storage ([Ceph][ceph]/[Rook][rook])
* Virtual LAN (also over multiple locations) ([Calico][calico])
* Load Balancing ([MetalLB][metallb])

## Preparations

### Servers

Acquire a couple of [servers][buy] as the basis for a cluster (`AX41-NVME`'s are working well for instance). Visit the [admin panel][admin] and name the servers appropriately.

```text
tc-k8s-node1
tc-k8s-node2
tc-k8s-node3
...

tc-k8s-master1
tc-k8s-master2
tc-k8s-worker1
tc-k8s-worker2
tc-k8s-worker3
...
```

Refer to the [reset procedure][reset] to properly initialize them.

### vSwitch

Create a [vSwitch][vswitch] and order an appropriate subnet (it may take a while to show up after the order). Give the vSwitch a name (i.e. `tc-k8s-net`) and assign this vSwitch to the servers.

Checkout the [docs][vswitch_docs] for help.

## Usage

Clone this repository, `cd` into it and download kubespray.

```bash
git submodule init && git submodule update
```

Create a Python virtual environment or similar.

```bash
# Optional
virtualenv -p python3 venv
```

Install dependencies required by Python and Ansible Glaxy.

```bash
pip install -r requirements.python.txt
ansible-galaxy install -r requirements.ansible.yml
```

> Note: [Mitogen][mitogen] does not work with ansible collections and the strategy must be changed (i.e. `strategy: linear`).

### Provisioning

Create a deployment environment inventory file for each cluster you want to manage.

```bash
cp hosts.example inventory/production.yml
cp hosts.example inventory/test.yml
cp hosts.example inventory/environment.yml
...

cp hosts.example inventory/production-01.yml
cp hosts.example inventory/production-02.yml
...

cp hosts.example inventory/production-helsinki.yml
cp hosts.example inventory/whatever.yml
```

Edit the inventory file with your server ip's and network information and customize everything to your needs.

```bash
# Manage a cluster
ansible-playbook cluster.init.yml -i inventory/environment.yml
ansible-playbook --become --become-user=root kubespray/cluster.yml -i inventory/environment.yml
ansible-playbook cluster.finish.yml -i inventory/environment.yml

# Run custom playbooks
ansible-playbook private-cluster.yml -i inventory/environment.yml
ansible-playbook private-test-cluster.yml -i inventory/environment.yml
ansible-playbook private-whatever-cluster.yml -i inventory/environment.yml
```

> Check [this][kubespray] out for more playbooks on cluster management.

### THORChain

In order for the cluster to operate as a node in the THORCHain network deploy as instructed [here][tc_deplyoing]. You can also refer to the [node-launcher repository][node-launcher], if necessary, or the THORChain [documentation][tc_docs] as a whole.

## Resetting the bare metal servers

This will install and use Ubuntu 20.04 on only one of the two internal NVMe drives. The unused ones will be used for persistent storage with ceph/rook. You can check the internal drive setup with `lsblk`. Change it accordingly in the command shown above when necessary.

### Manually

Visit the [console][admin] and put each server of the cluster into rescue mode. Then execute the following script.

```bash
installimage -a -r no -i images/Ubuntu-2004-focal-64-minimal.tar.gz -p /:ext4:all -d nvme0n1 -f yes -t yes -n hostname
```

### Automatically

Create a pristine state by running the playbooks in sequence.

```bash
ansible-playbook server.rescue.yml -i inventory/environment.yml
ansible-playbook server.bootstrap.yml -i inventory/environment.yml
```

### Instantiation

Instantiate the servers.

```bash
ansible-playbook server.instantiate.yml -i inventory/environment.yml
```

## Administration

### File system

Deploy, use and remove the [Rook Toolbox](toolbox).

```bash
ansible-playbook cluster.toolbox.yml -i inventory/environment.yml

kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash

ansible-playbook cluster.toolbox.yml -e state=absent -i inventory/environment.yml
```

[reset]: #resetting-the-bare-metal-servers
[hetzner]: https://www.hetzner.com
[buy]: https://www.hetzner.com/dedicated-rootserver/matrix-ax
[admin]: https://robot.your-server.de/server
[vswitch]: https://robot.your-server.de/vswitch/index
[vswitch_docs]: https://docs.hetzner.com/robot/dedicated-server/network/vswitch
[k8s]: https://kubernetes.io
[kubespray]: https://kubespray.io/
[metallb]: https://metallb.universe.tf
[calico]: https://www.projectcalico.org
[ceph]: https://ceph.io
[rook]: https://rook.io
[toolbox]: https://rook.io/docs/rook/v1.4/ceph-toolbox.html
[mitogen]: https://mitogen.readthedocs.io/en/python3/ansible.html
[tc]: https://thorchain.org
[tc_docs]: https://docs.thorchain.org
[tc_deplyoing]: https://docs.thorchain.org/thornodes/kubernetes/deploying
[node-launcher]: https://gitlab.com/thorchain/devops/node-launcher
