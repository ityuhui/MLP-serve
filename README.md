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

### Start a POD to mount PV

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mlp-model-store-pod
spec:
  volumes:
    - name: mlp-model-store-volume
      persistentVolumeClaim:
        claimName: mlp-pv-claim
  containers:
    - name: mlp-model-store-container
      image: ubuntu
      command: [ "sleep" ]
      args: [ "infinity" ]
      volumeMounts:
        - mountPath: "/pv"
          name: mlp-model-store-volume
      resources:
        limits:
          memory: "1Gi"
          cpu: "1"
EOF
```

### Copy model to PV

```bash
kubectl cp MLP.mar model-store-pod:/pv/MLP.mar -c model-store-container
```

### Deploy `InferenceService` with the model on PVC

```bash
cat << EOF | kubectl apply -f -
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "mlp-inference-service"
spec:
  predictor:
    pytorch:
      storageUri: "pvc://mlp-pv-claim/MLP.mar"
EOF
```

