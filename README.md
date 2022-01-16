# OpenSource Kubernetes on Fedora Linux 35

### I Recenlty tried to installed OpenSource Kubernetes on Fedora 35, it was a pain, so I figured I'd share my process here. Enjoy!

### Prereq's
1. Configure an external load balancer for the api:6443 interface and direct this port to the three master (control-plane) nodes.
2. Setup DNS to resolve the api:6443 external interface - I used `api.redcloud.land`
3. Install 6x Fedora 35 VM's, 3 for the master nodes and 3 for the worker (compute) nodes.
    ##### Note: For the partitioning, I created it manually so that I would avoid a swap partition. With VM's of 120GB, I chose 2g for /boot/efi, 18g for /, and 200g for /var/lib (containers will go here).

### Install
1. Boot up all of the VM's and update their patches: \
`dnf update -y`
2. We're going to install CRI-O along with Kubernetes 1.23 on these machines, each machine will include kubeadm and kubelet; only the first master will include kubectl. After the install is complete, I'll use kubectl from my bastion/workstation.

    a. Create the file to load the modules needed for crio.
    ```
    cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
    overlay
    br_netfilter
    EOF
    ```
    b. Setup the sysctl params.
    ```
    cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF
    ```
3. Apply the firewall rules: \

    a. Master Nodes:
    ```
    sudo firewall-cmd --add-port=2379-2380/tcp --add-port=6443/tcp --add-port=10250/tcp  --add-port=10257/tcp --add-port=10259/tcp --permanent
    sudo firewall-cmd --reload
    ```
    b. Worker Nodes:
    ```
    sudo firewall-cmd --add-port=10250/tcp --add-port=80/tcp --add-port=443/tcp --add-port=30000-32767/tcp --permanent
    sudo firewall-cmd --reload
    ```
4. Remove the ZRAM swap from all nodes: \
    `dnf remove zram-generator-defaults -y`
5. Install CRI-O:
    ##### Note: You can search for your crio version with `sudo dnf module list cri-o`, at the time of writing this, CRI-O 1.22 was supported with Kubernetes 1.22 & 1.23, but CRI-O 1.23 wasn't published for Fedora 35.
    ```
    sudo dnf module enable cri-o:1.22 -y
    sudo dnf install cri-o -y
    sudo systemctl enable --now crio
    ```
6. Configure the Kuberentes Repo:
    ```
    cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    exclude=kubelet kubeadm kubectl
    EOF
    ```
7. Disable SELinux (It's not really supported with Kubernetes and Fedora):
    ```
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    ```

8. Install Kubernetes (kubeadm/kubelet (kubectl)):
    a. Master Nodes:
    ```
    sudo dnf install -y kubelet kubeadm apt-transport-https kubectl --disableexcludes=kubernetes
    sudo systemctl enable --now kubelet
    ```
    b. Worker Nodes:
    ```
    sudo dnf install -y kubelet kubeadm apt-transport-https --disableexcludes=kubernetes
    sudo systemctl enable --now kubelet
    ```

9. INSTALL FIRST MASTER!\
    `sudo kubeadm init --control-plane-endpoint "api.redcloud.land:6443" --ignore-preflight-errors=Swap --upload-certs`

    You should see "Your Kubernetes control-plane has initialized successfully!"

    CONGRATS! \
    You will be provided the required tokens and join command for each node type and instruction on how to generage your .kube/config file.

 10. Lastly, you'll need to install a Container Networking Interface (CNI) for the Pod network, I chose Weave Networks for this install. I've also used Calico; both work great, Weave is just super basic and simple:\
    `kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

### From here, you will have a clean working Kubernetes cluster, see the kubernetes.org's documentation pages for installing the 'dashboard UI' and other settings. 
If you found this helpful, please drop me a 'star'!
