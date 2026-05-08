# LLMの学習（???+LoRA）

## モデルのダウンロード
管理者nodeで以下のコマンドを実行してください．
```
mkdir -p /home4cluster/models

docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/pytorch:25.05-py3 \
  python3 -c "
from huggingface_hub import snapshot_download
snapshot_download(
    repo_id='Qwen/Qwen2.5-7B-Instruct',
    local_dir='/home4cluster/models/Qwen2.5-7B-Instruct'
)
print('Download complete')
"
```

## 1node-LoRAチューニング
LoRAチューニング用のスクリプトを作成します．
```
mkdir -p /home4cluster/lora_train

cat > /home4cluster/lora_train/train.py << 'EOF'
import os
import time
import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import get_peft_model, LoraConfig, TaskType
from torch.utils.data import Dataset, DataLoader, DistributedSampler

MODEL_PATH = "/home4cluster/models/Qwen2.5-7B-Instruct"

# ダミーデータセット
class DummyDataset(Dataset):
    def __init__(self, tokenizer, num_samples=100, seq_len=512):
        self.input_ids = torch.randint(0, tokenizer.vocab_size, (num_samples, seq_len))
        self.labels = self.input_ids.clone()

    def __len__(self):
        return len(self.input_ids)

    def __getitem__(self, idx):
        return {"input_ids": self.input_ids[idx], "labels": self.labels[idx]}

def main():
    dist.init_process_group(backend="nccl")
    rank = dist.get_rank()
    world_size = dist.get_world_size()
    torch.cuda.set_device(0)
    device = torch.device("cuda", 0)

    if rank == 0:
        print(f"world_size: {world_size}")
        print(f"Loading model from {MODEL_PATH}")

    tokenizer = AutoTokenizer.from_pretrained(MODEL_PATH)
    model = AutoModelForCausalLM.from_pretrained(
        MODEL_PATH,
        torch_dtype=torch.bfloat16,
    ).to(device)

    # LoRA設定
    lora_config = LoraConfig(
        task_type=TaskType.CAUSAL_LM,
        r=8,
        lora_alpha=16,
        target_modules=["q_proj", "v_proj"],
        lora_dropout=0.05,
    )
    model = get_peft_model(model, lora_config)
    if rank == 0:
        model.print_trainable_parameters()

    model = DDP(model, device_ids=[0])

    dataset = DummyDataset(tokenizer)
    sampler = DistributedSampler(dataset, num_replicas=world_size, rank=rank)
    dataloader = DataLoader(dataset, batch_size=1, sampler=sampler)

    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)

    # 学習ループ
    model.train()
    start = time.time()
    total_loss = 0
    steps = 0

    for batch in dataloader:
        input_ids = batch["input_ids"].to(device)
        labels = batch["labels"].to(device)
        outputs = model(input_ids=input_ids, labels=labels)
        loss = outputs.loss
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        total_loss += loss.item()
        steps += 1

    elapsed = time.time() - start

    if rank == 0:
        print(f"steps: {steps}, avg_loss: {total_loss/steps:.4f}")
        print(f"elapsed: {elapsed:.1f}s")
        print(f"throughput: {steps * world_size / elapsed:.2f} samples/sec")

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
EOF
```
起動スクリプトの作成をします．
```
cat > /home4cluster/lora_train/run_1node.sh << 'EOF'
#!/bin/bash

ssh mprg@node15 "docker run --rm --gpus all --network host \
    --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
    -v /home4cluster:/home4cluster \
    -v /home/mprg/nccl-build:/home/mprg/nccl-build \
    -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \
    -e NCCL_SOCKET_IFNAME=enP7s7 \
    -e NCCL_IB_DISABLE=1 \
    -e NCCL_NET=Socket \
    nvcr.io/nvidia/pytorch:25.05-py3 \
    bash -c 'pip install -q peft transformers && torchrun \
        --nnodes=1 \
        --nproc_per_node=1 \
        --master_addr=localhost \
        --master_port=29500 \
        /home4cluster/lora_train/train.py'"
EOF
chmod +x /home4cluster/lora_train/run_1node.sh
```
```
bash /home4cluster/lora_train/run_1node.sh
```

## 2node-LoRAチューニング(RJ45)
```
cat > /home4cluster/lora_train/run_2node_rj45.sh << 'EOF'
#!/bin/bash

MASTER_ADDR="10.0.0.15"
MASTER_PORT="29500"
NNODES=2
NODES=("node15" "node16")

for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}
    RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl-build:/home/mprg/nccl-build \
        -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enP7s7 \
        -e NCCL_IB_DISABLE=1 \
        -e NCCL_NET=Socket \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers && torchrun \
            --nnodes=$NNODES \
            --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR \
            --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train.py'" &
done
wait
EOF
chmod +x /home4cluster/lora_train/run_2node_rj45.sh
```

## 2node-LoRAチューニング(QSFP)
```
cat > /home4cluster/lora_train/run_2node_qsfp.sh << 'EOF'
#!/bin/bash

MASTER_ADDR="10.0.1.1"
MASTER_PORT="29500"
NNODES=2
NODES=("node15" "node16")

for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}
    RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl-build:/home/mprg/nccl-build \
        -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enp1s0f0np0 \
        -e NCCL_IB_DISABLE=0 \
        -e NCCL_IB_HCA=rocep1s0f0 \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers && torchrun \
            --nnodes=$NNODES \
            --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR \
            --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train.py'" &
done
wait
EOF
chmod +x /home4cluster/lora_train/run_2node_qsfp.sh
bash /home4cluster/lora_train/run_2node_qsfp.sh
```
