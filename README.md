## Setup Persistent Volume for Models

```bash
# Create persistent volume claim
oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: llama-31
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
      claimName: llama-31
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

```bash
#deploy rhaiis

oc apply -f rhaiis/openshift
```


# GuideLLM

```bash
# Create persistent volume claim
oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tokenizer
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
      claimName: tokenizer
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
