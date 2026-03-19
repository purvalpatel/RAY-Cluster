
## Ray + Kubernetes setup 
Add repository and install:
```BASH
helm repo add ray https://ray-project.github.io/kuberay-helm/
helm repo update

helm install kuberay-operator ray/kuberay-operator \
  --namespace ray-system \
  --create-namespace
```

verify:
```
kubectl get pods -n ray-system
```

Set label:
```
kubectl label node mdgp054 nvidia=h100
kubectl label node mdgp053 nvidia=h100
kubectl label node mdgp033 nvidia=h200
```
check labels:
```
kubectl get nodes --show-labels
```
## Automated distributed scheduling.
### Create PV
```
mkdir /data/kubernetes-nfs-storage/hf-cache-ray/
```
Create `pv.yaml`
```YAML
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hf-cache-pv-ray
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.10.110.53       # replace with your NFS server IP
    path: /data/kubernetes-nfs-storage/hf-cache-ray
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
```
### Create PVC: `pvc.yaml`
```YAML
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hf-cache-pvc-ray
  namespace: ray-llm
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: ""
  volumeName: hf-cache-pv-ray
```
### Create namespace:
```
kubectl create ns ray-llm
```

### Job for download model : ` download-job.yaml`
```YAML
apiVersion: batch/v1
kind: Job
metadata:
  name: download-llama
  namespace: ray-llm
spec:
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: downloader
        image: python:3.11-slim
        command:
        - bash
        - -c
        - |
          pip install -q huggingface_hub && \
          python3 -c "
          from huggingface_hub import snapshot_download
          snapshot_download(
            repo_id='meta-llama/Llama-3.3-70B-Instruct',
            local_dir='/root/.cache/huggingface/hub/models--meta-llama--Llama-3.3-70B-Instruct',
            token='$(HF_TOKEN)'
          )
          "
        env:
        - name: HF_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-token
              key: HUGGING_FACE_HUB_TOKEN
        - name: HF_HOME
          value: /root/.cache/huggingface
        resources:
          requests:
            cpu: "2"
            memory: 8Gi
          limits:
            cpu: "4"
            memory: 16Gi
        volumeMounts:
        - name: hf-cache
          mountPath: /root/.cache/huggingface
      volumes:
      - name: hf-cache
        persistentVolumeClaim:
          claimName: hf-cache-pvc-ray       # your ray-llm PVC name
```

### Cluster Ray Cluster:
Create ray-cluster.yaml
```YAML
apiVersion: ray.io/v1
kind: RayService
metadata:
  name: ray-serve-llm
  namespace: ray-llm
spec:
  serviceUnhealthySecondThreshold: 1800     # 30 min — 70B takes long to load
  deploymentUnhealthySecondThreshold: 1800
  serveConfigV2: |
    applications:
    - name: llms
      import_path: ray.serve.llm:build_openai_app
      route_prefix: "/"
      args:
        llm_configs:
        - model_loading_config:
            model_id: meta-llama/Llama-3.3-70B-Instruct
            model_source: meta-llama/Llama-3.3-70B-Instruct
          engine_kwargs:
            dtype: bfloat16
            max_model_len: 8192
            gpu_memory_utilization: 0.90
            tensor_parallel_size: 4        # GPUs per node
            pipeline_parallel_size: 3      # number of worker nodes
          deployment_config:
            autoscaling_config:
              min_replicas: 1
              max_replicas: 1
              target_ongoing_requests: 32
            max_ongoing_requests: 64

  rayClusterConfig:
    rayVersion: "2.52.0"

    headGroupSpec:
      rayStartParams:
        num-cpus: "0"
        num-gpus: "0"
        dashboard-host: "0.0.0.0"
      template:
        spec:
          containers:
          - name: ray-head
            image: rayproject/ray-llm:2.52.0-py311-cu128
            ports:
            - containerPort: 6379
              name: gcs
            - containerPort: 8265
              name: dashboard
            - containerPort: 8000
              name: serve
            - containerPort: 8080
              name: metrics
            - containerPort: 10001
              name: client
            env:
            - name: HUGGING_FACE_HUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token
                  key: hf_token
            - name: HF_HOME
              value: /home/ray/.cache/huggingface
            resources:
              requests:
                cpu: "4"
                memory: "16Gi"
              limits:
                cpu: "8"
                memory: "32Gi"
            volumeMounts:
            - name: hf-cache
              mountPath: /home/ray/.cache/huggingface
          volumes:
          - name: hf-cache
            persistentVolumeClaim:
              claimName: hf-cache-pvc-ray

    workerGroupSpecs:
    - groupName: gpu-group
      replicas: 1
      minReplicas: 1
      maxReplicas: 1
      numOfHosts: 3
      rayStartParams:
        num-gpus: "4"
      template:
        spec:
          containers:
          - name: ray-worker
            image: rayproject/ray-llm:2.52.0-py311-cu128
            env:
            - name: HUGGING_FACE_HUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token
                  key: hf_token
            - name: HF_HOME
              value: /home/ray/.cache/huggingface
            - name: NCCL_DEBUG
              value: "WARN"
            resources:
              requests:
                cpu: "32"
                memory: "120Gi"
                nvidia.com/gpu: "4"
              limits:
                cpu: "32"
                memory: "120Gi"
                nvidia.com/gpu: "4"
            volumeMounts:
            - name: hf-cache
              mountPath: /home/ray/.cache/huggingface
            - name: shm
              mountPath: /dev/shm
          volumes:
          - name: hf-cache
            persistentVolumeClaim:
              claimName: hf-cache-pvc-ray
          - name: shm
            emptyDir:
              medium: Memory
              sizeLimit: 60Gi
          tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    ray.io/group: gpu-group
                topologyKey: kubernetes.io/hostname

```
### Explaination:

#### Head node - manager
- 8 cores and 32Gi RAM
- Dashboard port - 8265

#### worker group - do the work
- group of similar worker machines (pods) that do the actual computing work.
- With this configuration, Kubernetes will start 3 Ray worker pods.
- here we are creating group for h100 GPU nodes.
- each ray-worker pod requires 4 GPUs.
- pods created:
```
ray-worker-1
ray-worker-2
ray-worker-3
```
Kubernetes scheduler will place them on nodes with available GPUs. <br>
Example:
```
Node1
 ├ GPU0 GPU1 GPU2 GPU3 → ray-worker-1
 └ GPU4 GPU5 GPU6 GPU7 → ray-worker-2

Node2
 ├ GPU0 GPU1 GPU2 GPU3 → ray-worker-3
```
So your Ray cluster will effectively see:
```
3 worker pods × 4 GPUs = 12 GPUs total
```
- Ray will then distribute tasks across those GPUs.
apply:
```BASH
kubectl apply -f ray-cluster.yaml
```
![alt text](image-7.png)

Verify GPU availability in Ray:
```
kubectl exec -it ray-cluster-head-t96jf -n ray-system -- bash
(base) ray@ray-cluster-head-t96jf:~$ ray status
```

### Get Ray clusters:
```
kubectl get rayclusters -A
```

### Ray Dashboard Node port service:
```
apiVersion: v1
kind: Service
metadata:
  name: ray-dashboard
  namespace: ray-system
spec:
  type: NodePort
  selector:
    ray.io/node-type: head
  ports:
  - name: dashboard
    port: 8265
    targetPort: 8265
    nodePort: 30265
  - name: serve
    port: 8080
    targetPort: 8080
    nodePort: 30080

```
> open in browser: http://10.10.110.53:30265/#/overview

![alt text](image-8.png)

## Now Deploy models into ray cluster.
Earlier flow was:
```
User
 ↓
Istio
 ↓
Kubernetes Deployment
 ↓
vLLM pod
 ↓
GPU
```
With Ray it becomes:
```
User
 ↓
Istio
 ↓
Ray Serve
 ↓
Ray workers
 ↓
GPUs across nodes
```
LLama3.3 70B on Ray Cluster:

![alt text](image-9.png)


- Kuberay-operator is running in ray-system namespace. and workers and head is running in ray-llama namespace.

check operator watches all namespace:
```
kubectl describe deployment kuberay-operator -n ray-system | grep -A5 "Environment"
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:

## none means ALL namespaces.
```
## Test Job schedule:
```
kubectl -n ray-llm exec -it llama70b-raycluster-head-nbmns -- bash

(base) ray@llama70b-raycluster-head-nbmns:~$ ray job submit --address http://localhost:8265 -- python -c "import ray; ray.init(); print(ray.cluster_resources())"
```

- This job is showing in ray dashboard.

### Create `gpu_test.py` on host:
```Python
import ray
import torch
import socket
import os
import time

ray.init(address="auto")

@ray.remote(num_gpus=1)
def gpu_task(i):
    host = socket.gethostname()
    gpu = os.environ["CUDA_VISIBLE_DEVICES"]

    print(f"Task {i} running on host {host} GPU {gpu}")

    x = torch.randn(8000,8000, device="cuda")

    for _ in range(60):
        x = x @ x
        time.sleep(1)

    return host


tasks = [gpu_task.remote(i) for i in range(16)]
print(ray.get(tasks))
```

### Copy gpu_test.py into head:
```
kubectl -n ray-llm cp test.py llama70b-raycluster-head-pnk4v:/tmp/
```
### install dependencies on head and worker
```
kubectl -n ray-llm exec -it <head> -- pip install torch
kubectl -n ray-llm exec -it <worker-1> -- pip install torch
kubectl -n ray-llm exec -it <worker-2> -- pip install torch
kubectl -n ray-llm exec -it <worker-3> -- pip install torch
```
### run python program on head:
```
kubectl -n ray-llm exec -it <head> -- python /tmp/gpu_test.py
```

> Ray automatically spreads the tasks across the cluster. you dont need to SSH into each node and manage GPU ID's.

> without Ray, you would need to manually manage GPU allocation and coordinate between nodes, restart failure.




### error :
OSError: PermissionError at /root/.cache when downloading meta-llama/Llama-3.3-70B-Instruct. Check cache directory permissions. Common causes: 1) another user is downloading the same model (please wait); 2) a previous download was canceled and the lock file needs manual removal.


#### find all lock files
```
kubectl run remove-locks --rm -it --restart=Never \
  --image=busybox \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "remove-locks",
        "image": "busybox",
        "command": ["sh", "-c", "find /cache -name *.lock -o -name tmp* | xargs ls -la 2>/dev/null && find /cache -name *.lock -delete && find /cache -name tmp* -delete && echo DONE"],
        "volumeMounts": [{"name": "hf-cache", "mountPath": "/cache"}]
      }],
      "volumes": [{"name": "hf-cache", "persistentVolumeClaim": {"claimName": "hf-cache-pvc-ray"}}]
    }
  }' -n ray-llm
```
#### verify:
```
  kubectl run check-cache --rm -it --restart=Never \
  --image=busybox \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "check-cache",
        "image": "busybox",
        "command": ["sh", "-c", "ls -la /cache/hub/ && echo --- && find /cache -name *.lock 2>/dev/null | wc -l && echo lock files"],
        "volumeMounts": [{"name": "hf-cache", "mountPath": "/cache"}]
      }],
      "volumes": [{"name": "hf-cache", "persistentVolumeClaim": {"claimName": "hf-cache-pvc-ray"}}]
    }
  }' -n ray-llm
```

#### Verify GPU utilization:
```
watch -n 30 'kubectl get pods -n ray-llm --no-headers | grep worker | awk "{print \$1}
" | \
  xargs -I {} sh -c "echo === {} === && kubectl exec {} -n ray-llm -- \
  nvidia-smi --query-gpu=memory.used,memory.total --format=csv,noheader 2>/dev/null"'
```
