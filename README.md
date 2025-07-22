## Red Hat AI Inference Server - RHAIIS and GuideLLM on OpenShift

This repository will guide you through deploying RHAIIS and GuideLLM on OpenShift.  

Before we deploy the components, we will first create two persistent volumes one for the model weights to be used by RHAIIS, and the other one to store the tokenizer for GuideLLM and the Whisper Benchmark image

These instructions will use a model located in local-models/llama, downloaded from RedHatAI/Meta-Llama-3.1-8B-Instruct-quantized.w4a16

## Setup Persistent Volume for model weights

This persistent volume will store the model weights, follow these instructions to create the PVC and copy the model weights from your local machine

```bash
# Create persistent volume claim
oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: llama-31-8b-instruct-w4a16
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
EOF
```
```bash
# Create temporary pod to copy model
oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: model-copy-pod
spec:
  containers:
  - name: model-copy
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: models
      mountPath: /mnt/models
  volumes:
  - name: models
    persistentVolumeClaim:
      claimName: llama-31-8b-instruct-w4a16
EOF
```
```bash
oc wait --for=condition=Ready pod/model-copy-pod
```
```bash
# Copy model from local machine
oc rsync ./local-models/llama/ model-copy-pod:/mnt/models/
```
```bash
# Clean up
oc delete pod model-copy-pod
```

# Tokenizer PVC
This persistent volume will store the tokenizer to be used by guidellm, follow these instructions to create the PVC and copy the tokenizer from your local machine

```bash
# Create persistent volume claim
oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: llama-31-8b-instruct-w4a16-tokenizer
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```
```bash
# Create temporary pod to copy model
oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: tokenizer-copy-pod
spec:
  containers:
  - name: tokenizer-copy
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: tokenizer
      mountPath: /mnt/tokenizer
  volumes:
  - name: tokenizer
    persistentVolumeClaim:
      claimName: llama-31-8b-instruct-w4a16-tokenizer
EOF
```
```bash
oc wait --for=condition=Ready pod/tokenizer-copy-pod
```
```bash
# Copy model from local machine
oc cp ./local-models/llama/tokenizer.json tokenizer-copy-pod:/mnt/tokenizer/
oc cp ./local-models/llama/config.json tokenizer-copy-pod:/mnt/tokenizer/
```
```bash
# Clean up
oc delete pod tokenizer-copy-pod
```

## Check Resource Limits

Before deploying RHAIIS, check if there are any ResourceQuotas or LimitRanges in your namespace that might restrict resource allocation for the model serving pod:

```bash
# Check ResourceQuotas
oc get resourcequotas -o yaml

# Check LimitRanges  
oc get limitranges -o yaml

# Check current resource usage
oc describe resourcequotas
oc describe limitranges
```

If you see restrictive limits on CPU, memory, or GPU resources, you may need to:
- Request quota increases from your cluster administrator
- Modify the limits in your namespace
- Override resource requests/limits in your Helm values

## Deploy RHAIIS

There are two options to deploy RHAIIS, by applying the objects in the rhaiis/openshift folder, or by using Helm charts.  The Helm charts option is more flexible because we can change things like the model and pvc names.

To deploy RHAIIS configured for the RedHatAI/Meta-Llama-3.1-8B-Instruct-quantized.w4a16 model, run:

```bash
oc apply -f rhaiis/openshift
```

To deploy using the Helm chart, run:

```bash
helm install llama-31-8b-instruct-w4a16 ./rhaiis/helm \
--set model.pvcName=llama-31-8b-instruct-w4a16 
```

Any of the Helm chart values can be overidden for example:

* model.pvcName
* model.servedModelName
* resources.gpu

Configuring these values should allow for diffenent models, with different GPU requirements, and model weight persistent volumes.

## Uninstall RHAIIS

To uninstall the simple openshift deployment run:

```bash
oc delete -f rhaiis/openshift
```

To uninstall using helm, run:

```bash
helm uninstall llama-31-8b-instruct-w4a16 
```


## Deploy GuideLLM
There are two options to deploy GuideLLM,  by applying the objects in the guidellm/openshift folder, or by using Helm charts.  The Helm charts option is more flexible because we can change things like the model and pvc names.

To deploy using OpenShift objects run:

```bash
oc apply -f guidellm/openshift
```

To deploy using the Helm chart, run:

```bash
helm install guidellm ./guidellm/helm \
--set benchmark.target=http://llama-31-8b-instruct-w4a16-rhaiis:8000 \
--set benchmark.model=meta-llama-3.1-8B-instruct-quantized.w4a16 \
--set storage.tokenizerPvcName=llama-31-8b-instruct-w4a16-tokenizer 
```

Any of the Helm chart values can be overidden for example:

* benchmark.maxRequests
* benchmark.rateType

Configuring these values should allow for diffenent models, with different GPU requirements, and model weight persistent volumes.

To uninstall the simple openshift deployment run:

```bash
oc delete -f guidellm/openshift
```

To uninstall using helm run:

```bash
helm uninstall guidellm 
```

## Deploy Whisper Benchmark
There are two options to deploy the Whisper benchmark,  by applying the objects in the whisper-benchmark/openshift folder, or by using Helm charts.  The Helm charts option is more flexible because we can change configuration values.

To deploy using OpenShift objects run:

```bash
oc apply -f whisper-benchmark/openshift
```

To deploy using the Helm chart, run:

```bash
helm install whisper-benchmark ./whisper-benchmark/helm \
--set benchmark.openaiApiBaseUrl=http://whisper-large-v2-rhaiis:8000 \
--set benchmark.targetModelName=openai/whisper-large-v2 \
--set storage.modelPvcName=whisper-tokenizer
```

Any of the Helm chart values can be overidden for example:

* benchmark.concurrentRequests
* benchmark.language
* resources.requests.memory
* resources.limits.memory

To uninstall the simple openshift deployment run:

```bash
oc delete -f whisper-benchmark/openshift
```

To uninstall using helm run:

```bash
helm uninstall whisper-benchmark 
```

## Results

Once the GuideLLM tests have completed you will see results shown like:

```bash
================================================================================
Metadata           | Request Stats                   | Out Tok/sec | Tot Tok/sec | Req Latency (sec) 
                   |                                   |            |             | mean
                   |                                   |            |             | median | p99
                   | TTFT (ms)                         | ITL (ms)   | TPOT (ms)   |
Benchmark          | Per Second   | Concurrency        | mean       | mean        | mean 
                   |              |                    | median     | p99         | mean   | median | p99   | mean   | median | p99
-------------------|--------------|--------------------|------------|-------------|--------|--------|-------|--------|--------|-------
synchronous        | 0.85         | 1.00               | 108.4      | 325.4       | 1.18
                   |              |                    | 1.18       | 1.18        | 29.7   | 29.4   | 37.0  | 9.1    | 9.1    | 9.1
throughput         | 19.31        | 92.45              | 2495.8     | 7489.3      | 4.74
                   |              |                    | 4.73       | 4.91        | 988.2  | 950.5  | 1879.2| 29.5   | 29.2   | 37.2
constant@3.15      | 3.13         | 4.04               | 400.4      | 1201.5      | 1.29
                   |              |                    | 1.29       | 1.36        | 37.2   | 35.7   | 95.3  | 9.9    | 9.9    | 10.1
constant@5.46      | 5.40         | 7.54               | 690.5      | 2072.0      | 1.40
                   |              |                    | 1.39       | 1.64        | 45.3   | 34.2   | 235.4 | 10.6   | 10.7   | 11.0
constant@7.77      | 7.56         | 11.41              | 967.4      | 2902.8      | 1.51
                   |              |                    | 1.50       | 1.76        | 45.3   | 36.1   | 197.5 | 11.5   | 11.5   | 12.5
constant@10.08     | 9.76         | 17.89              | 1249.7     | 3749.8      | 1.83
                   |              |                    | 1.84       | 2.13        | 61.0   | 39.4   | 260.1 | 13.9   | 14.1   | 15.5
constant@12.38     | 11.79        | 23.21              | 1508.7     | 4526.9      | 1.97
                   |              |                    | 1.97       | 2.33        | 62.0   | 40.9   | 275.2 | 15.0   | 15.2   | 17.4
constant@14.69     | 13.85        | 31.94              | 1772.6     | 5319.2      | 2.31
                   |              |                    | 2.37       | 2.77        | 86.3   | 46.2   | 342.8 | 17.5   | 18.3   | 20.8
constant@17.00     | 15.41        | 40.82              | 1972.2     | 5917.7      | 2.65
                   |              |                    | 2.80       | 3.37        | 119.8  | 51.4   | 490.9 | 19.9   | 21.6   | 24.5
constant@19.31     | 17.01        | 55.03              | 2177.3     | 6533.4      | 3.23
                   |              |                    | 3.24       | 4.26        | 166.4  | 61.5   | 569.8 | 24.2   | 25.1   | 31.2
================================================================================

```

Once the Whisper Benchmarking tool is complete you will see results e.g.

```bash
WER: 100.0
Total Requests: 479
Mean TTFT: 142.9556276326482
95th Percentile TTFT: 309.7570 ms
Mean ITL: 142.9556276326482
95th Percentile ITL: 309.7570 ms
Average Latency: 0.1430 seconds
Estimated req_Throughput: 216.48 requests/s
Estimated Throughput: 2164.80 tok/s
Benchmark finished. Final WER: 100.00%
```