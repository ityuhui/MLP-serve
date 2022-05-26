# MLP-serve

## Environment

Linux x86_64
e.g. `Ubuntu 20.04.4 LTS x86_64`

## Setup a Kubernetes cluster

For experimentation only:

```bash
mkdir -p /mnt/torchserve

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

## Prepare storage for the model

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

### Start a pod to mount PV

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

# Copy the model
mkdir -p /mnt/torchserve/mlp/model-store
cp MLP.mar /mnt/torchserve/mlp/model-store/

# Configure the model
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
kubectl exec torchserve-model-store-pod -c torchserve-model-store-container -- cat /pv/mlp/config/config.properties
kubectl exec torchserve-model-store-pod -c torchserve-model-store-container -- ls -lh /pv/mlp/model-store/MLP.mar
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

## Get the url and port of the inference service

```bash
INGRESS_GATEWAY_SERVICE=$(kubectl get svc --namespace istio-system --selector="app=istio-ingressgateway" --output jsonpath='{.items[0].metadata.name}')
kubectl port-forward --namespace istio-system svc/${INGRESS_GATEWAY_SERVICE} 8080:80
# start another terminal
export INGRESS_HOST=localhost
export INGRESS_PORT=8080
```

## Run a prediction with the [input.json](input.json)

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
> POST /v1/models/MLP:predict HTTP/1.1
> Host: mlp-inference-service.default.example.com
> User-Agent: curl/7.68.0
> Accept: */*
> Content-Length: 750
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 750 out of 750 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 2312
< content-type: application/json; charset=UTF-8
< date: Thu, 26 May 2022 10:16:32 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 38
<
{"predictions": [[45.20157241821289, 45.120811462402344, 45.14750289916992, 45.221412658691406, 45.13792037963867, 45.09972381591797, 45.13405990600586, 45.12519454956055, 45.04914855957031, 45.135047912597656, 45.04569625854492, 45.01668930053711, 44.990211486816406, 45.082637786865234, 45.087223052978516, 44.827232360839844, 44.948974609375, 44.96574020385742, 44.98857879638672, 44.95081329345703, 44.92815017700195, 44.912109375, 44.89902877807617, 44.89009094238281, 44.778282165527344, 44.88908004760742, 44.93364334106445, 44.74320983886719, 44.959938049316406, 44.9059944152832, 44.741477966308594, 44.75467300415039, 44.65039825439453, 44.68185806274414, 44.707916259765625, 44.662864685058594, 44.6770133972168, 44.56578826904297, 44.737850189208984, 44.6454963684082, 44.60885238647461, 44.6480712890625, 44.737449645996094, 44.61884689331055, 44.527400970458984, 44.57012176513672, 44.50341796875, 44.487762451171875, 44.620697021484375, 44.462074279785156, 44.48261642456055, 44.46953582763672, 44.37877655029297, 44.302799224853516, 44.348087310791016, 44.3648681640625, 44.32415008544922, 44.3484992980957, 44.31337356567383, 44.19575119018555], [44.55545425415039, 44.47001647949219, 44.45315933227539, 44.523746490478516, 44.468177795410156, 44.45429229736328, 44.48317337036133, 44.46806716918945, 44.39951705932617, 44.46620178222656, 44.39321517944336, 44.33964157104492, 44.342437744140625, 44.41317367553711, 44.4006462097168, 44.188011169433594, 44.29681396484375, 44.32097244262695, 44.35060501098633, 44.30091094970703, 44.28242874145508, 44.25028991699219, 44.255680084228516, 44.20143127441406, 44.12403869628906, 44.27590560913086, 44.28573989868164, 44.11004638671875, 44.30231475830078, 44.25871276855469, 44.111473083496094, 44.09855270385742, 44.02037811279297, 44.030372619628906, 44.05878829956055, 44.01447296142578, 44.01744842529297, 43.95254135131836, 44.09235382080078, 44.03144073486328, 43.95957946777344, 44.01929473876953, 44.09056854248047, 43.95699691772461, 43.91106414794922, 43.938697814941406, 43.88779067993164, 43.84574508666992, 43.9615478515625, 43.82283020019531, 43.814483642578125, 43.80765151977539, 43.76226043701172, 43.66639709472656, 43.73677444458008, 43.74308395385742, 43.70634460449219, 43.73307800292969, 43.67891311645508, 43.57352828979492]]}
```

## Clean the environment

```bash
kind delete cluster
```
