---
date: '2025-04-21T17:35:27+05:30'
draft: false
title: 'Setting Up Lxd Lxc and K3s With a Basic Nginx Test Service'
---

This guide details how to set up a lightweight Kubernetes (K3s) cluster using LXD and LXC containers on an Ubuntu 24.04 server, followed by deploying a basic NGINX test service with NGINX Ingress. These steps were tested on an Oracle Cloud Free Tier instance named `maata-paarvati` with 4 CPUs and 24 GB RAM.

## Prerequisites

- An Ubuntu 24.04 server (e.g., Oracle Cloud Free Tier instance).
- Root or sudo access.
- Network configured to allow TCP ports 80, 443, and 6443.

---

## 1. Install LXD

1. **Install LXD via Snap**:

   ```bash
   sudo snap install lxd
   ```

2. **Initialize LXD**:

   - Use default settings for a simple setup:

     ```bash
     sudo lxd init --auto
     ```

   - This creates a `dir` storage pool (`default`) and a bridge network (`lxdbr0`).

3. **Add User to LXD Group**:

   ```bash
   sudo usermod -aG lxd ubuntu
   newgrp lxd
   ```

4. **Verify LXD**:

   ```bash
   lxc list
   ```

   - Should display an empty table initially.

---

## 2. Create a Kubernetes Profile for LXC Containers

1. **Create the Profile Configuration**:

   - Save this as `k8s-profile.yaml`:

     ```yaml
     config:
       limits.cpu: "2"
       limits.memory: 2GB
       limits.memory.swap: "false"
       linux.kernel_modules: ip_tables,ip6_tables,nf_nat,overlay,br_netfilter
       raw.lxc: |
         lxc.apparmor.profile=unconfined
         lxc.cap.drop=
         lxc.cgroup.devices.allow=a
         lxc.mount.auto=proc:rw sys:rw
       security.privileged: "true"
       security.nesting: "true"
       security.syscalls.intercept.mknod: "true"
       security.syscalls.intercept.setxattr: "true"
     description: LXD profile for Kubernetes
     devices:
       eth0:
         name: eth0
         nictype: bridged
         parent: lxdbr0
         type: nic
       kmsg:
         path: /dev/kmsg
         source: /dev/console
         type: unix-char
       root:
         path: /
         pool: default
         type: disk
     name: k8s
     used_by: []
     ```

   - This matches your `lxc config show kmaster --expanded` output, excluding volatile and proxy settings added later.

2. **Apply the Profile**:

   ```bash
   lxc profile create k8s
   lxc profile edit k8s < k8s-profile.yaml
   ```

3. **Verify Profile**:

   ```bash
   lxc profile show k8s
   ```

---

## 3. Launch LXC Containers with Ubuntu 24.04

1. **Initialize Containers**:

   ```bash
   lxc init ubuntu:24.04 kmaster --profile k8s
   lxc init ubuntu:24.04 kworker1 --profile k8s
   lxc init ubuntu:24.04 kworker2 --profile k8s
   ```

2. **Start Containers**:

   ```bash
   lxc start kmaster
   lxc start kworker1
   lxc start kworker2
   ```

3. **Verify Containers**:

   ```bash
   lxc list
   ```

   - Example output (IPs will vary):

     ```
     +---------+---------+----------------------+------+-----------+-----------+
     |  NAME   |  STATE  |         IPV4         | IPV6 |   TYPE    | SNAPSHOTS |
     +---------+---------+----------------------+------+-----------+-----------+
     | kmaster | RUNNING | 10.177.108.101 (eth0)|      | CONTAINER | 0         |
     | kworker1| RUNNING | 10.177.108.79 (eth0) |      | CONTAINER | 0         |
     | kworker2| RUNNING | 10.177.108.57 (eth0) |      | CONTAINER | 0         |
     +---------+---------+----------------------+------+-----------+-----------+
     ```

---

## 4. Install K3s and Join the Cluster

1. **Prepare** `kmaster`:

   ```bash
   lxc exec kmaster -- bash -c "apt update && apt install -y curl iptables"
   ```

2. **Install K3s on** `kmaster`:

   - Disable Traefik to use NGINX Ingress later:

     ```bash
     lxc exec kmaster -- bash -c "curl -sfL https://get.k3s.io | sh -s - server --disable=traefik --write-kubeconfig-mode 644"
     ```

3. **Get Node Token**:

   ```bash
   lxc exec kmaster -- cat /var/lib/rancher/k3s/server/node-token
   ```

   - Save the token (e.g., `K10...::server:...`).

4. **Prepare and Join Worker Nodes**:

   - For `kworker1`:

     ```bash
     lxc exec kworker1 -- bash -c "apt update && apt install -y curl iptables"
     lxc exec kworker1 -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://10.177.108.101:6443 K3S_TOKEN=<node-token> sh -"
     ```

   - For `kworker2`:

     ```bash
     lxc exec kworker2 -- bash -c "apt update && apt install -y curl iptables"
     lxc exec kworker2 -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://10.177.108.101:6443 K3S_TOKEN=<node-token> sh -"
     ```

5. **Verify Cluster**:

   ```bash
   lxc exec kmaster -- k3s kubectl get nodes
   ```

   - Expected:

     ```
     NAME       STATUS   ROLES                  AGE   VERSION
     kmaster    Ready    control-plane,master   5m    v1.32.3+k3s1
     kworker1   Ready    <none>                 2m    v1.32.3+k3s1
     kworker2   Ready    <none>                 1m    v1.32.3+k3s1
     ```

6. **Set Up Host Access**:

   - Install `kubectl` on the host:

     ```bash
     sudo snap install kubectl --classic
     ```

   - Copy kubeconfig:

     ```bash
     lxc file pull kmaster/etc/rancher/k3s/k3s.yaml ~/.kube/config
     ```

   - Edit `~/.kube/config`, replace `127.0.0.1` with `10.177.108.101`:

     ```yaml
     server: https://10.177.108.101:6443
     ```

   - Test:

     ```bash
     kubectl get nodes
     ```

7. **Add Port Forwarding for K3s API**:

   ```bash
   sudo iptables -t nat -A PREROUTING -p tcp --dport 6443 -j DNAT --to-destination 10.177.108.101:6443
   sudo iptables -t nat -A POSTROUTING -j MASQUERADE
   sudo sysctl -w net.ipv4.ip_forward=1
   ```

---

## 5. Set Up NGINX Ingress and Deploy a Test Service

1. **Install Helm**:

   ```bash
   curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
   sudo bash get_helm.sh
   ```

2. **Install NGINX Ingress**:

   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo update
   helm install ingress-nginx ingress-nginx/ingress-nginx \
     --namespace ingress-nginx \
     --create-namespace \
     --set controller.service.type=LoadBalancer
   ```

3. **Add Proxy Devices for HTTP/HTTPS**:

   - Based on your `lxc config show kmaster --expanded`:

     ```bash
     lxc config device add kmaster http proxy listen=tcp:0.0.0.0:80 connect=tcp:127.0.0.1:80
     lxc config device add kmaster https proxy listen=tcp:0.0.0.0:443 connect=tcp:127.0.0.1:443
     ```

4. **Add Port Forwarding for HTTP/HTTPS**:

   - Get NGINX service IP:

     ```bash
     kubectl get svc -n ingress-nginx
     ```

     - Example: `10.177.108.x`.

   - Forward ports:

     ```bash
     sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.177.108.x:80
     sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination 10.177.108.x:443
     sudo iptables -t nat -A POSTROUTING -j MASQUERADE
     ```

5. **Deploy a Basic NGINX Test Service**:

   - Create `nginx-test.yaml`:

     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx-test
       namespace: default
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: nginx-test
       template:
         metadata:
           labels:
             app: nginx-test
         spec:
           containers:
           - name: nginx
             image: nginx:latest
             ports:
             - containerPort: 80
     ---
     apiVersion: v1
     kind: Service
     metadata:
       name: nginx-test
       namespace: default
     spec:
       ports:
       - port: 80
         targetPort: 80
         protocol: TCP
       selector:
         app: nginx-test
     ---
     apiVersion: networking.k8s.io/v1
     kind: Ingress
     metadata:
       name: nginx-test
       namespace: default
       annotations:
         nginx.ingress.kubernetes.io/rewrite-target: /
     spec:
       ingressClassName: nginx
       rules:
       - host: nginx-test.local
         http:
           paths:
           - path: /
             pathType: Prefix
             backend:
               service:
                 name: nginx-test
                 port:
                   number: 80
     ```

   - Apply:

     ```bash
     kubectl apply -f nginx-test.yaml
     ```

6. **Test the NGINX Service**:

   - Update `/etc/hosts` on your local machine (temporary workaround since `nginx-test.local` isn’t a real domain):

     ```bash
     echo "<public-ip> nginx-test.local" | sudo tee -a /etc/hosts
     ```

     - Use your Oracle Cloud instance’s public IP (e.g., `curl ifconfig.me`).

   - Test:

     ```bash
     curl http://nginx-test.local
     ```

     - Should return the NGINX welcome page HTML.

---

## Conclusion

You now have a K3s cluster running inside LXC containers managed by LXD, with NGINX Ingress and a basic NGINX test service accessible at `http://nginx-test.local`. This setup leverages your Oracle Cloud Free Tier instance efficiently, with Traefik disabled and NGINX handling ingress traffic.
