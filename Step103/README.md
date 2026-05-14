# ステップ103：LLMの学習（Qwen2.5-7B-Instruct+フルファインチューニング+tatsu-lab/alpaca）

## フルファインチューニング学習速度比較結果

### 実験設定

| 項目 | 値 |
|------|-----|
| モデル | Qwen2.5-7B-Instruct |
| データセット | tatsu-lab/alpaca（1,000件） |
| シーケンス長 | 512トークン |
| バッチサイズ | 1 |
| 学習可能パラメータ | 7,615,616,512（全体の100%） |
| ウォームアップステップ | 5 |
| seed | 42 |
| フレームワーク | PyTorch DDP |
| Dockerイメージ | nvcr.io/nvidia/pytorch:25.05-py3 |

### 結果

| 構成 | throughput | avg_step_time | elapsed | 対1node比 |
|------|-----------|---------------|---------|----------|
| 1node | 0.43 samples/sec | 2.303s | 2,308s（38分） | 1.00x |
| 2node-RJ45 | 0.13 samples/sec | 14.818s | 7,344s（122分） | 0.30x |
| 2node-QSFP | 0.20 samples/sec | 9.952s | 4,935s（82分） | 0.47x |
| 4node-RJ45 | 0.19 samples/sec | 21.331s | 5,231s（87分） | 0.30x |

### 考察

- 2nodeにするとスループットが**1nodeより低下**するという逆転現象が発生
- 原因：フルFTでは全パラメータ（7.6B）の勾配をall-reduceで同期するため、約15GB/stepの通信が発生し、ネットワークがボトルネックになる
- 2node-RJ45は1nodeの**0.30倍**、2node-QSFPは**0.47倍**のスループット
- QSFPはRJ45と比べて約**1.5倍**高速であり、フルFTではLoRAと異なりQSFPの効果が明確に現れている
- フルFTにおいてはQSFPスイッチを用いた高速なall-reduce通信環境が必要
- 

## ステップ103.1：学習スクリプトの作成
フルファインチューニング用の学習スクリプトを作成します．
管理者nodeで以下のコマンドを実行してください．
```
cat > /home4cluster/lora_train/train_alpaca_full.py << 'EOF'
import os
import time
import random
import csv
import torch
import numpy as np
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from transformers import AutoModelForCausalLM, AutoTokenizer
from torch.utils.data import Dataset, DataLoader, DistributedSampler
from datasets import load_from_disk

# 設定
MODEL_PATH = "/home4cluster/models/Qwen2.5-7B-Instruct"
DATASET_PATH = "/home4cluster/datasets/alpaca"
NUM_SAMPLES = 1000
SEQ_LEN = 512
WARMUP_STEPS = 5
BATCH_SIZE = 1
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
    log_name = f"{world_size}node_{CONNECT_TYPE}_full_{timestamp}"

    if rank == 0:
        os.makedirs(LOG_DIR, exist_ok=True)
        print(f"=== 実験設定 ===")
        print(f"world_size   : {world_size}")
        print(f"connect_type : {CONNECT_TYPE}")
        print(f"train_type   : full fine-tuning")
        print(f"num_samples  : {NUM_SAMPLES}")
        print(f"seq_len      : {SEQ_LEN}")
        print(f"batch_size   : {BATCH_SIZE}")
        print(f"seed         : {SEED}")
        print(f"log_name     : {log_name}")
        print()

    tokenizer = AutoTokenizer.from_pretrained(MODEL_PATH)
    tokenizer.pad_token = tokenizer.eos_token

    model = AutoModelForCausalLM.from_pretrained(
        MODEL_PATH,
        dtype=torch.bfloat16,
    ).to(device)

    if rank == 0:
        total_params = sum(p.numel() for p in model.parameters())
        print(f"trainable params: {total_params:,} || all params: {total_params:,} || trainable%: 100.0000")
        print()

    model = DDP(model, device_ids=[0])

    dataset = AlpacaDataset(tokenizer)
    sampler = DistributedSampler(dataset, num_replicas=world_size, rank=rank, seed=SEED)
    dataloader = DataLoader(dataset, batch_size=BATCH_SIZE, sampler=sampler)
    total_steps = len(dataloader)

    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-5)

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
                with open(step_csv_path, "a", newline="") as f:
                    writer = csv.writer(f)
                    writer.writerow([steps, f"{loss_val:.4f}", f"{step_time:.3f}"])

                if (steps - WARMUP_STEPS) % LOG_INTERVAL == 0:
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

        summary_csv_path = f"{LOG_DIR}/results_full.csv"
        file_exists = os.path.exists(summary_csv_path)
        with open(summary_csv_path, "a", newline="") as f:
            writer = csv.writer(f)
            if not file_exists:
                writer.writerow([
                    "timestamp", "world_size", "connect_type",
                    "train_type", "num_samples", "seq_len", "batch_size", "seed",
                    "measured_steps", "avg_loss", "elapsed",
                    "throughput", "avg_step_time"
                ])
            writer.writerow([
                time.strftime("%Y-%m-%d %H:%M:%S"),
                world_size, CONNECT_TYPE, "full",
                NUM_SAMPLES, SEQ_LEN, BATCH_SIZE, SEED,
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

## ステップ103.2：学習
### 1node
```
cat > /home4cluster/lora_train/run_full_1node.sh << 'EOF'
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
        /home4cluster/lora_train/train_alpaca_full.py'"
EOF
chmod +x /home4cluster/lora_train/run_full_1node.sh
bash /home4cluster/lora_train/run_full_1node.sh
```



### 2node-rj45
```
cat > /home4cluster/lora_train/run_full_2node_rj45.sh << 'EOF'
#!/bin/bash
MASTER_ADDR="10.0.0.15"; MASTER_PORT="29500"; NNODES=2
NODES=("node15" "node16")
for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}; RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl-build:/home/mprg/nccl-build \
        -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enP7s7 \
        -e NCCL_IB_DISABLE=1 \
        -e NCCL_NET=Socket \
        -e CONNECT_TYPE=rj45 \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers datasets && torchrun \
            --nnodes=$NNODES --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train_alpaca_full.py'" &
done; wait
EOF
chmod +x /home4cluster/lora_train/run_full_2node_rj45.sh
bash /home4cluster/lora_train/run_full_2node_rj45.sh
```


### 2node-qsfp
node17とnode18で実行します．
```
cat > /home4cluster/lora_train/run_full_2node_qsfp.sh << 'EOF'
#!/bin/bash
MASTER_ADDR="10.0.2.1"; MASTER_PORT="29500"; NNODES=2
NODES=("node17" "node18")
for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}; RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl-build:/home/mprg/nccl-build \
        -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enp1s0f0np0 \
        -e NCCL_IB_DISABLE=0 \
        -e NCCL_IB_HCA=rocep1s0f0 \
        -e CONNECT_TYPE=qsfp \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers datasets && torchrun \
            --nnodes=$NNODES --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train_alpaca_full.py'" &
done; wait
EOF
chmod +x /home4cluster/lora_train/run_full_2node_qsfp.sh
bash /home4cluster/lora_train/run_full_2node_qsfp.sh
```


### 4node-rj45
```
cat > /home4cluster/lora_train/run_full_4node_rj45.sh << 'EOF'
#!/bin/bash
MASTER_ADDR="10.0.0.15"; MASTER_PORT="29500"; NNODES=4
NODES=("node15" "node16" "node17" "node18")
for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}; RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl-build:/home/mprg/nccl-build \
        -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enP7s7 \
        -e NCCL_IB_DISABLE=1 \
        -e NCCL_NET=Socket \
        -e CONNECT_TYPE=rj45 \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers datasets && torchrun \
            --nnodes=$NNODES --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train_alpaca_full.py'" &
done; wait
EOF
chmod +x /home4cluster/lora_train/run_full_4node_rj45.sh
bash /home4cluster/lora_train/run_full_4node_rj45.sh
```





