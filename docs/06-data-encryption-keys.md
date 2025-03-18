# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

Create the `encryption-config.yaml` encryption config file:

```bash
envsubst < configs/encryption-config.yaml \
  > encryption-config.yaml
```

Copy the `encryption-config.yaml` encryption config file to each controller instance:

```bash
FILE="encryption-config.yaml"

# Ensure the file exists before proceeding
if [[ ! -f "$FILE" ]]; then
  echo "ERROR: $FILE does not exist. Aborting."
  exit 1
fi

for host in $(awk '$3 ~ /plane/ {print $3}' machines.txt); do
  echo "Copying $FILE to $host..."

  scp "$FILE" "root@$host:~/"
  if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to copy $FILE to $host"
    continue
  fi

  echo "Successfully copied $FILE to $host"
done
```

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
