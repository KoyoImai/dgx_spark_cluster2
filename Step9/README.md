# ステップ9：LLMの学習（Qwen2.5-7B-Instruct+LoRA+tatsu-lab/alpaca）-バッチサイズ変更-

## バッチサイズ変更実験結果（LoRA, 1node）

| batch_size | throughput | avg_step_time | elapsed | peak_vram | 対bs=1比 |
|-----------|-----------|---------------|---------|-----------|---------|
| 1 | 2.12 samples/sec | 0.453s | 469s | 14.45GB | 1.00x |
| 2 | 2.33 samples/sec | 0.840s | 425s | 14.63GB | 1.10x |
| 4 | 2.60 samples/sec | 1.520s | 377s | 14.99GB | 1.23x |
| 8 | 2.66 samples/sec | 2.995s | 362s | 15.66GB | 1.26x |
| 16 | 2.79 samples/sec | 5.720s | 333s | 17.04GB | 1.32x |

## バッチサイズ変更実験結果（LoRA, 2node-RJ45）

| batch_size | throughput | avg_step_time | elapsed | peak_vram | 対1node-bs1比 |
|-----------|-----------|---------------|---------|-----------|--------------|
| 1 | 4.19 samples/sec | 0.460s | 236s | 14.45GB | 1.98x |
| 2 | 4.67 samples/sec | 0.840s | 210s | 14.63GB | 2.20x |
| 4 | 5.14 samples/sec | 1.539s | 187s | 14.99GB | 2.42x |
| 8 | - | - | - | - | - |
| 16 | - | - | - | - | - |


## ステップ9.1：学習スクリプトの作成
管理者nodeで以下のコマンドを実行し，学習スクリプトを作成してください．
```
cat > /home4cluster/lora_train/train_alpaca_bs.py << 'EOF'
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
BATCH_SIZE = int(os.environ.get("BATCH_SIZE", "1"))
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
    log_name = f"{world_size}node_{CONNECT_TYPE}_bs{BATCH_SIZE}_{timestamp}"

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

    step_csv_path = f"{LOG_DIR}/{log_name}_steps.csv"
    if rank == 0:
        with open(step_csv_path, "w", newline="") as f:
            writer = csv.writer(f)
            writer.writerow(["step", "loss", "step_time", "vram_gb"])

    model.train()
    total_loss = 0.0
    steps = 0
    start = None
    step_times = []
    peak_vram = 0.0

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

        vram_gb = torch.cuda.memory_allocated() / 1024**3
        if vram_gb > peak_vram:
            peak_vram = vram_gb

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
                    writer.writerow([steps, f"{loss_val:.4f}",
                                     f"{step_time:.3f}", f"{vram_gb:.2f}"])

                if (steps - WARMUP_STEPS) % LOG_INTERVAL == 0:
                    print(f"step {steps:4d}/{total_steps} | "
                          f"loss: {loss_val:.4f} | "
                          f"step_time: {step_time:.3f}s | "
                          f"throughput: {world_size * BATCH_SIZE / step_time:.2f} samples/sec | "
                          f"vram: {vram_gb:.1f}GB")

    torch.cuda.synchronize()
    elapsed = time.time() - start
    measured_steps = steps - WARMUP_STEPS
    avg_loss = total_loss / measured_steps
    throughput = measured_steps * BATCH_SIZE * world_size / elapsed
    avg_step_time = sum(step_times) / len(step_times)

    if rank == 0:
        print(f"\n=== 結果 ===")
        print(f"steps        : {steps} (warmup: {WARMUP_STEPS}, measured: {measured_steps})")
        print(f"avg_loss     : {avg_loss:.4f}")
        print(f"elapsed      : {elapsed:.1f}s")
        print(f"throughput   : {throughput:.2f} samples/sec")
        print(f"avg_step_time: {avg_step_time:.3f}s/step")
        print(f"peak_vram    : {peak_vram:.2f}GB")

        summary_csv_path = f"{LOG_DIR}/results_bs.csv"
        file_exists = os.path.exists(summary_csv_path)
        with open(summary_csv_path, "a", newline="") as f:
            writer = csv.writer(f)
            if not file_exists:
                writer.writerow([
                    "timestamp", "world_size", "connect_type",
                    "num_samples", "seq_len", "batch_size", "lora_r", "seed",
                    "measured_steps", "avg_loss", "elapsed",
                    "throughput", "avg_step_time", "peak_vram_gb"
                ])
            writer.writerow([
                time.strftime("%Y-%m-%d %H:%M:%S"),
                world_size, CONNECT_TYPE,
                NUM_SAMPLES, SEQ_LEN, BATCH_SIZE, LORA_R, SEED,
                measured_steps, f"{avg_loss:.4f}", f"{elapsed:.1f}",
                f"{throughput:.2f}", f"{avg_step_time:.3f}",
                f"{peak_vram:.2f}"
            ])
        print(f"\nステップログ : {step_csv_path}")
        print(f"サマリーログ : {summary_csv_path}")

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
EOF
```

## ステップ9.2：1nodeでの学習
### 起動スクリプトの作成
```
for BS in 1 2 4 8 16 32; do
cat > /home4cluster/lora_train/run_bs${BS}_1node.sh << EOF
#!/bin/bash
ssh mprg@node15 "docker run --rm --gpus all --network host \\
    --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \\
    -v /home4cluster:/home4cluster \\
    -v /home/mprg/nccl-build:/home/mprg/nccl-build \\
    -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \\
    -e NCCL_SOCKET_IFNAME=enP7s7 \\
    -e NCCL_IB_DISABLE=1 \\
    -e NCCL_NET=Socket \\
    -e CONNECT_TYPE=none \\
    -e BATCH_SIZE=${BS} \\
    nvcr.io/nvidia/pytorch:25.05-py3 \\
    bash -c 'pip install -q peft transformers datasets && torchrun \\
        --nnodes=1 --nproc_per_node=1 \\
        --master_addr=localhost --master_port=29500 \\
        /home4cluster/lora_train/train_alpaca_bs.py'"
EOF
chmod +x /home4cluster/lora_train/run_bs${BS}_1node.sh
done
echo "全スクリプト作成完了"
```

### 学習の実行
バッチサイズ1
```
# log: /home4cluster/lora_train/logs/1node_none_bs1_20260509_093756_steps.csv
bash /home4cluster/lora_train/run_bs1_1node.sh
```

バッチサイズ2
```
# /home4cluster/lora_train/logs/1node_none_bs2_20260509_094810_steps.csv
bash /home4cluster/lora_train/run_bs2_1node.sh
```

バッチサイズ4
```
# /home4cluster/lora_train/logs/1node_none_bs4_20260509_095732_steps.csv
bash /home4cluster/lora_train/run_bs2_1node.sh
```

バッチサイズ8
```
# /home4cluster/lora_train/logs/1node_none_bs8_20260509_100709_steps.csv
bash /home4cluster/lora_train/run_bs8_1node.sh
```

バッチサイズ16
```
# /home4cluster/lora_train/logs/1node_none_bs16_20260509_101627_steps.csv
bash /home4cluster/lora_train/run_bs16_1node.sh
```



## ステップ9.3：2node（RJ45）での学習
### 起動スクリプトの作成
```
for BS in 1 2 4 8 16; do
cat > /home4cluster/lora_train/run_bs${BS}_2node_rj45.sh << EOF
#!/bin/bash
MASTER_ADDR="10.0.0.15"; MASTER_PORT="29500"; NNODES=2
NODES=("node15" "node16")
for i in "\${!NODES[@]}"; do
    NODE=\${NODES[\$i]}; RANK=\$i
    ssh mprg@\$NODE "docker run --rm --gpus all --network host \\
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \\
        -v /home4cluster:/home4cluster \\
        -v /home/mprg/nccl-build:/home/mprg/nccl-build \\
        -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \\
        -e NCCL_SOCKET_IFNAME=enP7s7 \\
        -e NCCL_IB_DISABLE=1 \\
        -e NCCL_NET=Socket \\
        -e CONNECT_TYPE=rj45 \\
        -e BATCH_SIZE=${BS} \\
        nvcr.io/nvidia/pytorch:25.05-py3 \\
        bash -c 'pip install -q peft transformers datasets && torchrun \\
            --nnodes=\$NNODES --nproc_per_node=1 \\
            --master_addr=\$MASTER_ADDR --master_port=\$MASTER_PORT \\
            --node_rank=\$RANK \\
            /home4cluster/lora_train/train_alpaca_bs.py'" &
done; wait
EOF
chmod +x /home4cluster/lora_train/run_bs${BS}_2node_rj45.sh
done
echo "全スクリプト作成完了"
```
### 学習の実行
バッチサイズ1
```
# /home4cluster/lora_train/logs/2node_rj45_bs1_20260509_104038_steps.csv
bash /home4cluster/lora_train/run_bs1_2node_rj45.sh
```

バッチサイズ2
```
# /home4cluster/lora_train/logs/2node_rj45_bs2_20260509_104850_steps.csv
bash /home4cluster/lora_train/run_bs1_2node_rj45.sh
```



バッチサイズ4
```
# /home4cluster/lora_train/logs/2node_rj45_bs4_20260509_105734_steps.csv
bash /home4cluster/lora_train/run_bs4_2node_rj45.sh
```


バッチサイズ8
```
#

```

