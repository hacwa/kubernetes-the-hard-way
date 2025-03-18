# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

Copy `etcd` binaries and systemd unit files to the `server` instance:

```bash
set -e

declare -A hosts
hosts["K8S-Control-Plane-01-VM"]="10.0.21.2"
hosts["K8S-Control-Plane-02-VM"]="10.0.21.3"
hosts["K8S-Control-Plane-03-VM"]="10.0.21.4"

for name in "${!hosts[@]}"; do
  host=${hosts[$name]}
  echo "Deploying etcd on $name ($host)..."

  # Copy etcd binaries and service file
  scp \
    /root/kubernetes-the-hard-way/downloads/etcd-v3.4.34-linux-amd64.tar.gz \
    "/root/kubernetes-the-hard-way/units/etcd-$name.service" \
    "root@$host:~/"

  if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to copy files to $host"
    continue
  fi

  # Execute commands remotely on each control plane
  ssh root@"$host" bash <<EOF
    echo "Extracting and moving etcd binaries..."
    tar -xvf etcd-v3.4.34-linux-amd64.tar.gz
    mv etcd-v3.4.34-linux-amd64/etcd* /usr/local/bin/

    echo "Creating etcd directories and setting permissions..."
    mkdir -p /etc/etcd /var/lib/etcd
    chmod 700 /var/lib/etcd

    echo "Copying required certificates..."
    cp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/kube-api-server.key /etc/kubernetes/pki/kube-api-server.crt /etc/etcd/

    echo "Moving etcd service file..."
    mv etcd-$name.service /etc/systemd/system/etcd.service

    echo "Reloading systemd and enabling etcd..."
    systemctl daemon-reload
    systemctl enable --now etcd || true
EOF

  echo "etcd setup completed on $name ($host)"
done

```



## Verification

List the etcd cluster members:

```bash
etcdctl member list
```

```text
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```

```bash
# Validate cluster health
echo ">>> Checking cluster health..."
ETCDCTL_API=3 etcdctl --endpoints="http://10.0.21.2:2379,http://10.0.21.3:2379,http://10.0.21.4:2379" endpoint health
```
```bash
# Validate cluster health
echo ">>> Checking cluster health..."
ETCDCTL_API=3 etcdctl --endpoints="http://10.0.21.2:2379" member list
```





Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
