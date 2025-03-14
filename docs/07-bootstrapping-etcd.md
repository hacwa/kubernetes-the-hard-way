# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

Copy `etcd` binaries and systemd unit files to the `server` instance:

```bash
for host in $(awk '{print $3}' machines.txt | grep Plane); do
scp \
  downloads/etcd-v3.4.34-linux-amd64.tar.gz \
  units/etcd.service \
  root@$host:~/ ;
done
```







The commands in this lab must be run on the `server` machine. Login to the `server` machine using the `ssh` command. Example:

```bash
ssh root@K8S-Control-Plane-01
ssh root@K8S-Control-Plane-02
ssh root@K8S-Control-Plane-03

```

## Bootstrapping an etcd Cluster

### Install the etcd Binaries

Extract and install the `etcd` server and the `etcdctl` command line utility:

```bash
{
  tar -xvf etcd-v3.4.34-linux-amd64.tar.gz
  mv etcd-v3.4.34-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server


```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/kube-api-server.key /etc/kubernetes/pki/kube-api-server.crt \
    /etc/etcd/
}
```

Create the `etcd.service` systemd unit file:

```bash


[Service]
Type=notify
Environment="ETCD_UNSUPPORTED_ARCH=amd64"
ExecStart=/usr/local/bin/etcd \
  --name K8S-Control-Plane-01 \
  --initial-advertise-peer-urls http://10.0.16.2:2380 \
  --listen-peer-urls http://10.0.16.2:2380 \
  --listen-client-urls http://10.0.16.2:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.16.2:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster K8S-Control-Plane-01=http://10.0.16.2:2380,K8S-Control-Plane-02=http://10.0.16.3:2380,K8S-Control-Plane-03=http://10.0.16.4:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

mv etcd.service /etc/systemd/system/

{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd &
}



[Service]
Type=notify
Environment="ETCD_UNSUPPORTED_ARCH=amd64"
ExecStart=/usr/local/bin/etcd \
  --name K8S-Control-Plane-02 \
  --initial-advertise-peer-urls http://10.0.16.3:2380 \
  --listen-peer-urls http://10.0.16.3:2380 \
  --listen-client-urls http://10.0.16.3:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.16.3:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster K8S-Control-Plane-01=http://10.0.16.2:2380,K8S-Control-Plane-02=http://10.0.16.3:2380,K8S-Control-Plane-03=http://10.0.16.4:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

mv etcd.service /etc/systemd/system/

{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd &
}



[Service]
Type=notify
Environment="ETCD_UNSUPPORTED_ARCH=amd64"
ExecStart=/usr/local/bin/etcd \
  --name K8S-Control-Plane-03 \
  --initial-advertise-peer-urls http://10.0.16.4:2380 \
  --listen-peer-urls http://10.0.16.4:2380 \
  --listen-client-urls http://10.0.16.4:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.16.4:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster K8S-Control-Plane-01=http://10.0.16.2:2380,K8S-Control-Plane-02=http://10.0.16.3:2380,K8S-Control-Plane-03=http://10.0.16.4:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

mv etcd.service /etc/systemd/system/

{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd &
}




```

### Start the etcd Server

```bash

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
#!/bin/bash

# Define cluster nodes and their IPs
NODES=(
  "K8S-Control-Plane-01 10.0.16.2"
  "K8S-Control-Plane-02 10.0.16.3"
  "K8S-Control-Plane-03 10.0.16.4"
)

# Common etcd cluster configuration
CLUSTER="K8S-Control-Plane-01=http://10.0.16.2:2380,K8S-Control-Plane-02=http://10.0.16.3:2380,K8S-Control-Plane-03=http://10.0.16.4:2380"

# Path to etcd binary archive
ETCD_ARCHIVE="etcd-v3.4.34-linux-amd64.tar.gz"

# Ensure the archive is available
if [[ ! -f "$ETCD_ARCHIVE" ]]; then
  echo "Error: $ETCD_ARCHIVE not found in the current directory."
  exit 1
fi

# Loop through each node to set up etcd
for NODE in "${NODES[@]}"; do
  NAME=$(echo "$NODE" | awk '{print $1}')
  IP=$(echo "$NODE" | awk '{print $2}')

  echo ">>> Setting up etcd on $NAME ($IP)..."

  # Transfer the etcd archive and extract it
  scp "$ETCD_ARCHIVE" "root@$IP:/root/"
  ssh root@$IP <<EOF
    set -e
    echo "Extracting etcd binaries..."
    tar -xvf /root/$ETCD_ARCHIVE
    mv etcd-v3.4.34-linux-amd64/etcd* /usr/local/bin/

    echo "Creating required directories..."
    mkdir -p /etc/etcd /var/lib/etcd
    chmod 700 /var/lib/etcd

    echo "Copying Kubernetes CA certificates..."
    cp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/kube-api-server.key /etc/kubernetes/pki/kube-api-server.crt /etc/etcd/

    echo "Creating systemd service file..."
    cat <<EOT > /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
Environment="ETCD_UNSUPPORTED_ARCH=amd64"
ExecStart=/usr/local/bin/etcd \\
  --name $NAME \\
  --initial-advertise-peer-urls http://$IP:2380 \\
  --listen-peer-urls http://$IP:2380 \\
  --listen-client-urls http://$IP:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls http://$IP:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster "$CLUSTER" \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
EOT

    echo "Starting etcd service..."
    systemctl daemon-reload
    systemctl enable etcd
    systemctl start etcd
    echo "Etcd setup completed on $NAME ($IP)."
EOF
done

echo ">>> All etcd nodes configured successfully!"

# Validate cluster health
echo ">>> Checking cluster health..."
ETCDCTL_API=3 etcdctl --endpoints="http://10.0.16.2:2379,http://10.0.16.3:2379,http://10.0.16.4:2379" endpoint health
ETCDCTL_API=3 etcdctl --endpoints="http://10.0.16.2:2379" member list
```





Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
