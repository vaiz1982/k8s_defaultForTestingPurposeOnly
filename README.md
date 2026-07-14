k8s configuration file only here! 



chmod +x k8s-setup.sh  
sudo NODE_ROLE=master ./k8s-setup.sh 


sudo NODE_ROLE=master bash -x ./k8s-setup.sh  // to see what is going on
Prepare host for k8s





This script automates the setup of a Kubernetes cluster on Ubuntu using:

kubeadm (to initialize/join the cluster)
CRI-O (container runtime)
Flannel (CNI networking plugin)

It can configure a machine as either a control plane (master) or a worker node.

Here's what it does, step by step.

1. Specify Bash interpreter
#!/usr/bin/env bash

Runs the script with Bash.

2. Exit on errors
set -euo pipefail

This makes the script safer.

-e → Exit immediately if any command fails.
-u → Error if an undefined variable is used.
pipefail → If any command in a pipeline fails, the pipeline fails.
3. Ensure the script is run as root
if [[ $EUID -ne 0 ]]; then
  echo "Please run as root (sudo $0)"
  exit 1
fi

Checks whether the effective user ID is root.

If not:

Please run as root

and exits.

4. Determine the node role
NODE_ROLE="${NODE_ROLE:-}"

Uses the environment variable if supplied.

Example:

sudo NODE_ROLE=master ./k8s-setup.sh

or

sudo NODE_ROLE=node ./k8s-setup.sh

If not supplied:

read -p "Node role (master/node): " NODE_ROLE

It asks:

Node role (master/node):
5. Update Ubuntu
apt update

Refreshes package information.

6. Install required packages
apt-get install curl apt-transport-https git iptables-persistent -y

Installs:

Package	Purpose
curl	Download files
apt-transport-https	Install packages over HTTPS
git	Git support
iptables-persistent	Save firewall rules
7. Create keyring directory
mkdir -p /etc/apt/keyrings

APT stores repository signing keys here.

8. Configure required kernel modules

Creates

/etc/modules-load.d/k8s.conf

with

br_netfilter
overlay

These modules are required for Kubernetes.

Then loads them immediately:

modprobe br_netfilter
modprobe overlay
9. Configure kernel networking

Creates

/etc/sysctl.d/k8s.conf

containing

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1

These settings:

Enable bridge traffic inspection
bridge-nf-call-iptables

Allows iptables to filter traffic passing through Linux bridges.

Kubernetes networking depends on this.

Enable IPv4 forwarding
net.ipv4.ip_forward=1

Allows packets to be routed between interfaces.

Required for Pods to communicate.

Apply settings:

sysctl --system
10. Install CRI-O

First add the repository key:

curl ...

Convert it into a GPG key:

gpg --dearmor

Add the repository:

https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main

Update packages:

apt update

Install:

apt install cri-o

CRI-O becomes the container runtime Kubernetes will use instead of Docker.

11. Install Kubernetes tools

Sets versions

VERSION="v1.34.0"
KUBERNETES_VERSION=1.34
12. Install crictl

Downloads

crictl-v1.34.0-linux-amd64.tar.gz

Extracts it to

/usr/local/bin

crictl is a CLI for interacting directly with CRI-compatible runtimes like CRI-O.

13. Add Kubernetes repository

Downloads repository key

curl ...

Adds

https://pkgs.k8s.io/core:/stable:/v1.34

Updates package lists.

14. Install Kubernetes binaries
apt install kubelet kubeadm kubectl

Installs:

kubelet

Runs on every node.

Responsible for:

starting pods
monitoring containers
talking to the API server
kubeadm

Cluster bootstrap tool.

Used for

kubeadm init

and

kubeadm join
kubectl

Command-line client.

Examples:

kubectl get nodes

kubectl get pods

kubectl apply
15. Enable CRI-O
systemctl enable crio

Starts CRI-O automatically after reboot.

Then

systemctl start crio.service

Starts it immediately.

16. If this is the master node
if [[ "$NODE_ROLE" == "master" ]]

Runs

kubeadm init --pod-network-cidr=10.244.0.0/16

This initializes the Kubernetes control plane.

It installs:

API Server
Scheduler
Controller Manager
etcd

The CIDR 10.244.0.0/16 matches Flannel's default network configuration.

17. Generate join command
INIT_CMD=$(kubeadm token create --print-join-command)

Produces something like:

kubeadm join 192.168.1.10:6443 \
--token abc.xyz \
--discovery-token-ca-cert-hash sha256:...

Worker nodes will use this command to join the cluster.

18. Ask for user information

Prompts:

HOME PATH:
USER NAME:
USER GROUP:

Example:

HOME PATH: /home/ubuntu

USER NAME: ubuntu

USER GROUP: ubuntu

This information is used to set up the kubeconfig file for the specified user.

19. Configure kubectl

Creates

~/.kube

Copies

/etc/kubernetes/admin.conf

to

~/.kube/config

Changes ownership:

chown ubuntu:ubuntu

This lets the specified user run kubectl without needing root.

20. Install Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.25.7/Documentation/kube-flannel.yml

This installs Flannel, which:

creates the pod network
assigns pod IPs
enables pod-to-pod communication across nodes
21. Print the join command
echo "$INIT_CMD"

Displays the command that worker nodes should execute.

22. If this is a worker node
elif [[ "$NODE_ROLE" == "node" ]]

Prompts:

JOIN COMMAND:

You paste the command printed by the master.

Example:

kubeadm join ...

Then

eval "$JOIN_CMD"

executes it, joining the node to the cluster.

23. Invalid role

If neither master nor node is supplied:

echo "Wrong role"
exit 1
Overall Flow
Start
   │
   ▼
Check root
   │
   ▼
Choose role (master/node)
   │
   ▼
Update Ubuntu
   │
   ▼
Install dependencies
   │
   ▼
Load kernel modules
   │
   ▼
Configure networking (sysctl)
   │
   ▼
Install CRI-O
   │
   ▼
Install kubeadm, kubelet, kubectl, crictl
   │
   ▼
Start CRI-O
   │
   ▼
      ┌──────────────┐
      │ MASTER?      │
      └──────┬───────┘
             │Yes
             ▼
     kubeadm init
             │
     Configure kubectl
             │
     Install Flannel
             │
     Print join command
             │
             ▼
            Done

             No
             │
             ▼
      Ask for join command
             │
      kubeadm join
             │
            Done

In summary, the script automates nearly all of the manual steps required to prepare an Ubuntu machine for Kubernetes: it configures the operating system, installs CRI-O and Kubernetes components, initializes a control plane (or joins a worker node), and, on the control plane, installs the Flannel networking plugin and outputs the command needed for worker nodes to join the cluster.
