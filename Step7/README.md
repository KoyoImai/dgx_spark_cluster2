# ステップ7：LLMの学習（???+LoRA）

## ステップ7.1：モデルのダウンロード
管理者nodeで以下のコマンドを実行してください．
```
mkdir -p /home4cluster/models

docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/pytorch:25.05-py3 \
  bash -c "pip install -q huggingface_hub && python3 -c \"
from huggingface_hub import snapshot_download
...
\""
print('Download complete')
"
```

## ステップ7.2：動作確認
### 1node-LoRAチューニング
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

### 2node-LoRAチューニング(RJ45)
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
```
bash /home4cluster/lora_train/run_2node_rj45.sh
```

### 2node-LoRAチューニング(QSFP)
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


### 4node-LoRAチューニング(RJ45)
```
cat > /home4cluster/lora_train/run_4node_rj45.sh << 'EOF'
#!/bin/bash

MASTER_ADDR="10.0.0.15"
MASTER_PORT="29500"
NNODES=4
NODES=("node15" "node16" "node17" "node18")

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
chmod +x /home4cluster/lora_train/run_4node_rj45.sh
bash /home4cluster/lora_train/run_4node_rj45.sh
```

## tatsu-lab/alpacaデータセットのダウンロード & 学習スクリプト作成
管理者nodeで以下のコマンドを実行してください．
```
docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/pytorch:25.05-py3 \
  bash -c "pip install -q huggingface_hub datasets && python3 -c \"
from datasets import load_dataset
dataset = load_dataset('tatsu-lab/alpaca', split='train')
dataset.save_to_disk('/home4cluster/datasets/alpaca')
print(f'Download complete: {len(dataset)} samples')
\""
```

```
cat > /home4cluster/lora_train/train_alpaca.py << 'EOF'
import os
import time
import random
import csv
import torch
import numpy as np
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import get_peft_model, LoraConfig, TaskType
from torch.utils.data import Dataset, DataLoader, DistributedSampler
from datasets import load_from_disk

# 設定
MODEL_PATH = "/home4cluster/models/Qwen2.5-7B-Instruct"
DATASET_PATH = "/home4cluster/datasets/alpaca"
NUM_SAMPLES = 1000
SEQ_LEN = 512
WARMUP_STEPS = 5
BATCH_SIZE = 1
LORA_R = 8
LORA_ALPHA = 16
SEED = 42
LOG_DIR = "/home4cluster/lora_train/logs"
CONNECT_TYPE = os.environ.get("CONNECT_TYPE", "unknown")
LOG_INTERVAL = 10

def set_seed(seed):
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)

class AlpacaDataset(Dataset):
    def __init__(self, tokenizer, num_samples=NUM_SAMPLES, seq_len=SEQ_LEN):
        dataset = load_from_disk(DATASET_PATH)
        self.samples = []
        for item in dataset.select(range(num_samples)):
            prompt = f"### Instruction:\n{item['instruction']}\n"
            if item['input']:
                prompt += f"### Input:\n{item['input']}\n"
            prompt += f"### Response:\n{item['output']}"
            encoded = tokenizer(
                prompt,
                max_length=seq_len,
                truncation=True,
                padding="max_length",
                return_tensors="pt"
            )
            self.samples.append({
                "input_ids": encoded["input_ids"].squeeze(),
                "labels": encoded["input_ids"].squeeze()
            })

    def __len__(self):
        return len(self.samples)

    def __getitem__(self, idx):
        return self.samples[idx]

def main():
    set_seed(SEED)
    dist.init_process_group(backend="nccl")
    rank = dist.get_rank()
    world_size = dist.get_world_size()
    torch.cuda.set_device(0)
    device = torch.device("cuda", 0)

    timestamp = time.strftime("%Y%m%d_%H%M%S")
    log_name = f"{world_size}node_{CONNECT_TYPE}_{timestamp}"

    if rank == 0:
        os.makedirs(LOG_DIR, exist_ok=True)
        print(f"=== 実験設定 ===")
        print(f"world_size   : {world_size}")
        print(f"connect_type : {CONNECT_TYPE}")
        print(f"num_samples  : {NUM_SAMPLES}")
        print(f"seq_len      : {SEQ_LEN}")
        print(f"batch_size   : {BATCH_SIZE}")
        print(f"lora_r       : {LORA_R}")
        print(f"seed         : {SEED}")
        print(f"log_name     : {log_name}")
        print()

    tokenizer = AutoTokenizer.from_pretrained(MODEL_PATH)
    tokenizer.pad_token = tokenizer.eos_token

    model = AutoModelForCausalLM.from_pretrained(
        MODEL_PATH,
        dtype=torch.bfloat16,
    ).to(device)

    lora_config = LoraConfig(
        task_type=TaskType.CAUSAL_LM,
        r=LORA_R,
        lora_alpha=LORA_ALPHA,
        target_modules=["q_proj", "v_proj"],
        lora_dropout=0.05,
    )
    model = get_peft_model(model, lora_config)
    if rank == 0:
        model.print_trainable_parameters()
        print()

    model = DDP(model, device_ids=[0])

    dataset = AlpacaDataset(tokenizer)
    sampler = DistributedSampler(dataset, num_replicas=world_size, rank=rank, seed=SEED)
    dataloader = DataLoader(dataset, batch_size=BATCH_SIZE, sampler=sampler)
    total_steps = len(dataloader)

    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)

    # ステップログ用CSVの準備
    step_csv_path = f"{LOG_DIR}/{log_name}_steps.csv"
    if rank == 0:
        with open(step_csv_path, "w", newline="") as f:
            writer = csv.writer(f)
            writer.writerow(["step", "loss", "step_time"])

    model.train()
    total_loss = 0.0
    steps = 0
    start = None
    step_times = []

    for batch in dataloader:
        input_ids = batch["input_ids"].to(device)
        labels = batch["labels"].to(device)

        step_start = time.time()
        outputs = model(input_ids=input_ids, labels=labels)
        loss = outputs.loss
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        torch.cuda.synchronize()
        step_time = time.time() - step_start

        steps += 1

        if steps == WARMUP_STEPS:
            torch.cuda.synchronize()
            start = time.time()

        if steps > WARMUP_STEPS:
            loss_val = loss.detach().item()
            total_loss += loss_val
            step_times.append(step_time)

            if rank == 0:
                # ステップログをCSVに保存
                with open(step_csv_path, "a", newline="") as f:
                    writer = csv.writer(f)
                    writer.writerow([steps, f"{loss_val:.4f}", f"{step_time:.3f}"])

                # LOG_INTERVALごとにprint
                if (steps - WARMUP_STEPS) % LOG_INTERVAL == 0:
                    measured = steps - WARMUP_STEPS
                    print(f"step {steps:4d}/{total_steps} | "
                          f"loss: {loss_val:.4f} | "
                          f"step_time: {step_time:.3f}s | "
                          f"throughput: {world_size / step_time:.2f} samples/sec")

    torch.cuda.synchronize()
    elapsed = time.time() - start
    measured_steps = steps - WARMUP_STEPS
    avg_loss = total_loss / measured_steps
    throughput = measured_steps * world_size / elapsed
    avg_step_time = sum(step_times) / len(step_times)

    if rank == 0:
        print(f"\n=== 結果 ===")
        print(f"steps        : {steps} (warmup: {WARMUP_STEPS}, measured: {measured_steps})")
        print(f"avg_loss     : {avg_loss:.4f}")
        print(f"elapsed      : {elapsed:.1f}s")
        print(f"throughput   : {throughput:.2f} samples/sec")
        print(f"avg_step_time: {avg_step_time:.3f}s/step")

        # サマリーCSVに保存
        summary_csv_path = f"{LOG_DIR}/results.csv"
        file_exists = os.path.exists(summary_csv_path)
        with open(summary_csv_path, "a", newline="") as f:
            writer = csv.writer(f)
            if not file_exists:
                writer.writerow([
                    "timestamp", "world_size", "connect_type",
                    "num_samples", "seq_len", "batch_size", "lora_r", "seed",
                    "measured_steps", "avg_loss", "elapsed",
                    "throughput", "avg_step_time"
                ])
            writer.writerow([
                time.strftime("%Y-%m-%d %H:%M:%S"),
                world_size, CONNECT_TYPE,
                NUM_SAMPLES, SEQ_LEN, BATCH_SIZE, LORA_R, SEED,
                measured_steps, f"{avg_loss:.4f}", f"{elapsed:.1f}",
                f"{throughput:.2f}", f"{avg_step_time:.3f}"
            ])
        print(f"\nステップログ : {step_csv_path}")
        print(f"サマリーログ : {summary_csv_path}")

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
EOF
```

## LoRLチューニング
### 1node
1nodeでのLoRAチューニング
```
cat > /home4cluster/lora_train/run_alpaca_1node.sh << 'EOF'
#!/bin/bash
ssh mprg@node15 "docker run --rm --gpus all --network host \
    --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
    -v /home4cluster:/home4cluster \
    -v /home/mprg/nccl-build:/home/mprg/nccl-build \
    -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \
    -e NCCL_SOCKET_IFNAME=enP7s7 \
    -e NCCL_IB_DISABLE=1 \
    -e NCCL_NET=Socket \
    -e CONNECT_TYPE=none \
    nvcr.io/nvidia/pytorch:25.05-py3 \
    bash -c 'pip install -q peft transformers datasets && torchrun \
        --nnodes=1 --nproc_per_node=1 \
        --master_addr=localhost --master_port=29500 \
        /home4cluster/lora_train/train_alpaca.py'"
EOF
chmod +x /home4cluster/lora_train/run_alpaca_1node.sh
bash /home4cluster/lora_train/run_alpaca_1node.sh
```


