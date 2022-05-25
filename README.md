# MLP-serve

## Setup a Kubernetes cluster

Only for experiment:
```bash
kind create cluster  --image kindest/node:v1.21.1
```

## Install KServe

Only for experiment:
```bash
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.7/hack/quick_install.sh" | bash
```

## Prepare storage for model

### Create directory

```bash
mkdir -p /mnt/mlp/data
chmod 777 /mnt/mlp/data
```

### Create PV and PVC

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mlp-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/mlp/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mlp-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```
