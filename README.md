## Ray Usage:
1. Distributed training
2. Resouse management
3. Fault tolerance
4. Pipeline orchestration
5. Large scale AI workloads

<img width="700" height="592" alt="image" src="https://github.com/user-attachments/assets/486b82d9-2e5a-4e59-a77b-85798ab1fe24" />

## POC on local machine:

#### single-node vLLM + Ray LLM container setup.
```
docker run -it --rm \
  --gpus all \
  --shm-size=60g \
  -p 8000:8000 \
  -p 8265:8265 \
  -e HUGGING_FACE_HUB_TOKEN=hf_xxxx \
  -e HF_HOME=/home/ray/.cache/huggingface \
  -v /data/kubernetes-nfs-storage/hf-cache:/home/ray/.cache/huggingface \
  rayproject/ray-llm:2.52.0-py311-cu128 \
  python3 -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.3-70B-Instruct \
    --tensor-parallel-size 4 \
    --dtype bfloat16 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.90 \
    --host 0.0.0.0 \
    --port 8000
  ```
#### 🔷 Step-by-step : Multi-node setup:
##### Run on master machine
```
sudo docker run -it --rm   --gpus all   --shm-size=60g   -p 6379:6379   -p 8265:8265   -p 10001:10001   -p 8000:8000   -e HUGGING_FACE_HUB_TOKEN=hf_YDBxXkvvvvvvvCfNBasxcKblWKXecJ   -v /data/kubernetes-nfs-storage/hf-cache-ray:/home/ray/.cache/huggingface   rayproject/ray-llm:2.52.0-py311-cu128   bash
```
inside container:
```
ray start --head \
  --port=6379 \
  --num-gpus=4 \
  --dashboard-host=0.0.0.0
```
you can check in browser: `http://10.10.110.53:8265/#/overview`

View status : `ray status` <br>
Terminate: `ray stop` <br>

##### Start worker nodes (Node1, Node2)
```
sudo docker run -it --rm \
  --gpus all   --network host \
  --shm-size=60g \
  -e HUGGING_FACE_HUB_TOKEN=hf_YDBxXkoaqaejOccmECfNxxxxxxxxx \
  -v /data/kubernetes-nfs-storage/hf-cache-nfs:/home/ray/.cache/huggingface \
  rayproject/ray-llm:2.52.0-py311-cu128 \
  bash
```
inside the container: (join the head node)
```
## head-node IP
ray start --address=10.10.110.xx:6379 --num-gpus=4
```
<img width="679" height="213" alt="image" src="https://github.com/user-attachments/assets/0819ca19-0fe1-43bc-bc20-3cfbf57fa5d9" />

Now check status:
```
ray status
```
<img width="676" height="560" alt="image" src="https://github.com/user-attachments/assets/4225fcdc-8a07-4e1e-a6c2-a2cdaddfc68a" />


Now run vLLM with multinode:
```
python3 -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3.3-70B-Instruct \
  --tensor-parallel-size 8 \
  --distributed-executor-backend ray \
  --dtype bfloat16 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90 \
  --host 0.0.0.0 \
  --port 8000
```
> Error : NCCL error: remote process exited or network error

**Note:** <br>

> This will breaks: <br>
Tensor Parallel (TP) for LLMs (like Llama-70B): <br>
Splits each layer across GPUs <br>
Requires ALL GPUs to talk every forward pass <br>
Uses NCCL all-reduce / broadcast heavily <br>

- This requires NVLink or Infiniband.
- Ethernets with 1G/10G will not work.
