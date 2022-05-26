# MLP-serve

## Setup a Kubernetes cluster

For experimentation only:

```bash
mkdir -p /mnt/torchserve
chmod 777 /mnt/torchserve

cat << EOF | kind create cluster --config -
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
    image: kindest/node:v1.21.1
    extraMounts:
      - hostPath: /mnt/torchserve
        containerPath: /torchserve
EOF
```

## Install KServe

For experimentation only:

```bash
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.7/hack/quick_install.sh" | bash
```

## Prepare storage for model

### Create PV and PVC

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: torchserve-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/torchserve"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: torchserve-pv-claim
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
  name: torchserve-model-store-pod
spec:
  volumes:
    - name: torchserve-model-store-volume
      persistentVolumeClaim:
        claimName: torchserve-pv-claim
  containers:
    - name: torchserve-model-store-container
      image: ubuntu
      command: [ "sleep" ]
      args: [ "infinity" ]
      volumeMounts:
        - mountPath: "/pv"
          name: torchserve-model-store-volume
      resources:
        limits:
          memory: "1Gi"
          cpu: "1"
EOF
```

### Copy model to PV

```bash

mkdir -p /mnt/torchserve/mlp/model-store
cp MLP.mar /mnt/torchserve/mlp/model-store/
mkdir /mnt/torchserve/mlp/config
cat > /mnt/torchserve/mlp/config/config.properties << EOF
inference_address=http://0.0.0.0:8085
management_address=http://0.0.0.0:8081
metrics_address=http://0.0.0.0:8082
enable_metrics_api=true
metrics_format=prometheus
number_of_netty_threads=4
job_queue_size=10
service_envelope=kfserving
model_store=/mnt/models/model-store
model_snapshot={"name":"startup.cfg","modelCount":1,"models":{"MLP":{"1.0":{"defaultVersion":true,"marName":"MLP.mar","minWorkers":1,"maxWorkers":5,"batchSize":1,"maxBatchDelay":5000,"responseTimeout":120}}}}
EOF

# Check
kubectl exec torchserve-model-store-pod -c torchserve-model-store-container -- ls /pv/
```

## Deploy `InferenceService` with the model on PVC

```bash
cat << EOF | kubectl apply -f -
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "mlp-inference-service"
spec:
  predictor:
    pytorch:
      storageUri: "pvc://torchserve-pv-claim/mlp"
EOF
```

## Run a prediction

```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice mlp-inference-service -o jsonpath='{.status.url}' | cut -d "/" -f 3)

MODEL_NAME=MLP
INPUT_PATH=@./input.json
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

Expected Output

```bash
*   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> POST /v1/models/sklearn-pvc:predict HTTP/1.1
> Host: sklearn-pvc.default.example.com
> User-Agent: curl/7.68.0
> Accept: */*
> Content-Length: 84
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 84 out of 84 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 23
< content-type: application/json; charset=UTF-8
< date: Mon, 20 Sep 2021 04:55:50 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 6
< 
* Connection #0 to host localhost left intact
{"predictions": [1, 1]}
```

## Clean

```bash
kind delete cluster
```
