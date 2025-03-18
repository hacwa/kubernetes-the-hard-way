# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes client configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), typically called kubeconfigs, which configure Kubernetes clients to connect and authenticate to Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `kubelet` and the `admin` user.

### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

> The following commands must be run in the same directory used to generate the SSL certificates during the [Generating TLS Certificates](04-certificate-authority.md) lab.

Generate a kubeconfig file for the node-0 worker node:

```bash
for host in $(cat machines.txt | awk '{print $3}' | grep WORKER); do
  kubectl config set-cluster hacwa\
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://K8S-LB-VIP-VM.hacwa.internal:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=hacwa\
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```

### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```bash
{
  kubectl config set-cluster hacwa\
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://K8S-LB-VIP-VM.hacwa.internal:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=hacwa\
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
}
```


### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the `kube-controller-manager` service:

```bash
{
  kubectl config set-cluster hacwa\
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://K8S-LB-VIP-VM.hacwa.internal:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=hacwa\
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```




### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the `kube-scheduler` service:

```bash
{
  kubectl config set-cluster hacwa\
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://K8S-LB-VIP-VM.hacwa.internal:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=hacwa\
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
}
```


### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:

```bash
{
  kubectl config set-cluster hacwa\
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://K8S-LB-VIP-VM.hacwa.internal:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=hacwa\
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default \
    --kubeconfig=admin.kubeconfig
}
```



## Distribute the Kubernetes Configuration Files

Copy the `kubelet` and `kube-proxy` kubeconfig files to the node-0 instance:

```bash

for host in $(awk '$3 ~ /WORKER/ {print $3}' machines.txt); do
  echo "Configuring kubelet and kube-proxy on $host..."
  
  # Ensure SSH connection works
  if ! ssh root@"$host" "mkdir -p /var/lib/kube-proxy /var/lib/kubelet"; then
    echo "ERROR: SSH connection failed for $host. Skipping..."
    continue
  fi

  FILES=(
    "kube-proxy.kubeconfig"
    "${host}.kubeconfig"
  )

  DESTS=(
    "/var/lib/kube-proxy/kubeconfig"
    "/var/lib/kubelet/kubeconfig"
  )

  for i in "${!FILES[@]}"; do
    if [[ ! -f "${FILES[$i]}" ]]; then
      echo "ERROR: File ${FILES[$i]} does not exist. Skipping..."
      continue
    fi

    scp "${FILES[$i]}" "root@$host:${DESTS[$i]}"
    if [[ $? -ne 0 ]]; then
      echo "ERROR: Failed to copy ${FILES[$i]} to $host"
      continue
    fi
  done

  echo "Setup completed for $host"
done
```

Copy the `kube-controller-manager` and `kube-scheduler` kubeconfig files to the controller instances:

```bash

FILES=(
  "admin.kubeconfig"
  "kube-controller-manager.kubeconfig"
  "kube-scheduler.kubeconfig"
)

# Ensure all required files exist before proceeding
for file in "${FILES[@]}"; do
  if [[ ! -f "$file" ]]; then
    echo "ERROR: File $file does not exist. Aborting."
    exit 1
  fi
done

for host in $(awk '$3 ~ /PLANE/ {print $3}' machines.txt); do
  echo "Copying kubeconfig files to $host..."

  for file in "${FILES[@]}"; do
    scp "$file" "root@$host:~/"
    if [[ $? -ne 0 ]]; then
      echo "ERROR: Failed to copy $file to $host"
      continue
    fi
  done

  echo "Successfully copied all files to $host"
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
