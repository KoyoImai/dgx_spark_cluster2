# ステップ10：LoRAランク変更実験

## 実験設定
 
| 項目 | 値 |
|------|-----|
| モデル | Qwen2.5-7B-Instruct |
| データセット | tatsu-lab/alpaca（1,000件） |
| シーケンス長 | 512トークン |
| バッチサイズ | 1 |
| LoRA rank (r) | **16 / 32 / 64**（変数） |
| LoRA alpha | r × 2（r=16→32, r=32→64, r=64→128） |
| 対象モジュール | q_proj, v_proj |
| ウォームアップステップ | 5 |
| seed | 42 |
| フレームワーク | PyTorch DDP + PEFT |
| Dockerイメージ | nvcr.io/nvidia/pytorch:25.05-py3 |
 
## LoRAランク別パラメータ詳細
 
### モデルアーキテクチャ（Qwen2.5-7B-Instruct）
 
| 項目 | 値 |
|------|-----|
| Transformer層数 | 28層 |
| hidden_size | 3,584 |
| num_attention_heads | 28（head_dim = 128） |
| num_key_value_heads | 4（GQA） |
| q_proj サイズ | 3,584 × 3,584 |
| v_proj サイズ | 3,584 × 512（GQAのためKVヘッド数が少ない） |
| 全パラメータ数 | 7,615,616,512 |
 
### LoRAパラメータ数の計算式
 
LoRAは各モジュールに `A`（r×in）と `B`（out×r）の2行列を追加します。
 
```
1層あたりのLoRAパラメータ:
  q_proj: r × (3584 + 3584) = r × 7,168
  v_proj: r × (3584 +  512) = r × 4,096
  1層合計:                   r × 11,264
 
全28層合計: r × 11,264 × 28 = r × 315,392
```
 
### ランク別パラメータ数・通信量一覧
 
| LoRA rank (r) | lora_alpha | 学習可能パラメータ数 | 全体比 | 通信量/step（BF16） |
|:---:|:---:|---:|---:|---:|
| r=8（既存・比較用） | 16 | 2,523,136 | 0.033% | 約 4.8 MB |
| **r=16** | **32** | **5,046,272** | **0.066%** | **約 9.6 MB** |
| **r=32** | **64** | **10,092,544** | **0.133%** | **約 19.2 MB** |
| **r=64** | **128** | **20,185,088** | **0.265%** | **約 38.5 MB** |
| **r=128** | **256** | **40,370,176** | **0.530%** | **約 77.0 MB** |

## LoRAランク変更実験結果（LoRA, 1node）

| LoRAランク | throughput | avg_step_time | elapsed | peak_vram | 対bs=1比 |
|-----------|-----------|---------------|---------|-----------|---------|
| 16 | 2.12 samples/sec | 0.454s | 469.8s |  |  |
| 32 | 2.09 samples/sec | 0.460s | 475.7s |  |  |
| 64 | 2.11 samples/sec | 0.456s | 472.6s |  |  |
| 128 | 2.03 samples/sec | 0.473s | 489.2s |  |  |


## LoRAランク変更実験結果（LoRA, 2node-RJ45）

| LoRAランク | throughput | avg_step_time | elapsed | peak_vram | 対bs=1比 |
|-----------|-----------|---------------|---------|-----------|---------|
| 16 | 4.06 samples/sec | 0.475s | 243.9s |  |  |
| 32 | 4.08 samples/sec | 0.473s | 242.4s |  |  |
| 64 | 3.91 samples/sec | 0.494s | 253.3s |  |  |
| 128 | 3.83 samples/sec | 0.503s | 258.4s |  |  |


## バッチサイズ変更実験結果（LoRA, 2node-QSFP）

| batch_size | throughput | avg_step_time | elapsed | peak_vram | 対1node-bs1比 |
|-----------|-----------|---------------|---------|-----------|--------------|
| 1 | 4.16 samples/sec | 0.463s | 237.7s |  |  |
| 2 | 4.78 samples/sec | 0.821s | 205.2s |  |  |
| 4 | 5.12 samples/sec | 1.546s | 187.6s |  |  |
| 8 | 5.27 samples/sec | 3.017s | 175.9s |  |  |
| 16 | 5.57 samples/sec | 5.724s | 155.0s |  |  |


## バッチサイズ変更実験結果（LoRA, 4node-RJ45）

| batch_size | throughput | avg_step_time | elapsed | peak_vram | 対1node-bs1比 |
|-----------|-----------|---------------|---------|-----------|--------------|
| 1 | 8.27 samples/sec | 0.466s | 118.5s |  |  |
| 2 | 9.27 samples/sec | 0.846s | 103.5s |  |  |
| 4 | 10.32 samples/sec | 1.533s | 89.9s |  |  |
| 8 | 10.74 samples/sec | 2.961s | 80.4s |  |  |
| 16 | 11.23 samples/sec | 5.679s | 62.7s |  |  |

 
## 実験設定
 
| 項目 | 値 |
|------|-----|
| モデル | Qwen2.5-7B-Instruct |
| データセット | tatsu-lab/alpaca（1,000件） |
| シーケンス長 | 512トークン |
| バッチサイズ | 1 |
| LoRA rank (r) | **16 / 32 / 64**（変数） |
| LoRA alpha | r × 2（r=16→32, r=32→64, r=64→128） |
| 対象モジュール | q_proj, v_proj |
| ウォームアップステップ | 5 |
| seed | 42 |
| フレームワーク | PyTorch DDP + PEFT |
| Dockerイメージ | nvcr.io/nvidia/pytorch:25.05-py3 |
 
## ステップ10.1：学習スクリプトの作成
 
管理者nodeで以下のコマンドを実行してください。
 
```bash
cat > /home4cluster/lora_train/train_alpaca_lora_rank.py << 'EOF'
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
LORA_R = int(os.environ.get("LORA_R", "8"))
LORA_ALPHA = LORA_R * 2   # alpha = 2r（スケーリング係数を一定に保つ）
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
    log_name = f"{world_size}node_{CONNECT_TYPE}_r{LORA_R}_{timestamp}"
 
    if rank == 0:
        os.makedirs(LOG_DIR, exist_ok=True)
        print(f"=== 実験設定 ===")
        print(f"world_size   : {world_size}")
        print(f"connect_type : {CONNECT_TYPE}")
        print(f"num_samples  : {NUM_SAMPLES}")
        print(f"seq_len      : {SEQ_LEN}")
        print(f"batch_size   : {BATCH_SIZE}")
        print(f"lora_r       : {LORA_R}")
        print(f"lora_alpha   : {LORA_ALPHA}")
        print(f"seed         : {SEED}")
        print(f"log_name     : {log_name}")
        print()
 
    tokenizer = AutoTokenizer.from_pretrained(MODEL_PATH)
    tokenizer.pad_token = tokenizer.eos_token
 
    model = AutoModelForCausalLM.from_pretrained(
        MODEL_PATH,
        torch_dtype=torch.bfloat16,
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
        print(f"lora_r       : {LORA_R}  lora_alpha: {LORA_ALPHA}")
        print(f"trainable    : see above")
        print(f"steps        : {steps} (warmup: {WARMUP_STEPS}, measured: {measured_steps})")
        print(f"avg_loss     : {avg_loss:.4f}")
        print(f"elapsed      : {elapsed:.1f}s")
        print(f"throughput   : {throughput:.2f} samples/sec")
        print(f"avg_step_time: {avg_step_time:.3f}s/step")
 
        summary_csv_path = f"{LOG_DIR}/results_lora_rank.csv"
        file_exists = os.path.exists(summary_csv_path)
        with open(summary_csv_path, "a", newline="") as f:
            writer = csv.writer(f)
            if not file_exists:
                writer.writerow([
                    "timestamp", "world_size", "connect_type",
                    "num_samples", "seq_len", "batch_size",
                    "lora_r", "lora_alpha", "seed",
                    "measured_steps", "avg_loss", "elapsed",
                    "throughput", "avg_step_time"
                ])
            writer.writerow([
                time.strftime("%Y-%m-%d %H:%M:%S"),
                world_size, CONNECT_TYPE,
                NUM_SAMPLES, SEQ_LEN, BATCH_SIZE,
                LORA_R, LORA_ALPHA, SEED,
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
 
---
 
## ステップ10.2：起動スクリプトの一括生成
 
管理者nodeで以下を実行してください。12本のスクリプトがまとめて生成されます。
 
```bash
for R in 16 32 64 128; do
 
# --- 1node ---
cat > /home4cluster/lora_train/run_r${R}_1node.sh << EOF
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
    -e LORA_R=${R} \\
    nvcr.io/nvidia/pytorch:25.05-py3 \\
    bash -c 'pip install -q peft transformers datasets && torchrun \\
        --nnodes=1 --nproc_per_node=1 \\
        --master_addr=localhost --master_port=29500 \\
        /home4cluster/lora_train/train_alpaca_lora_rank.py'"
EOF
chmod +x /home4cluster/lora_train/run_r${R}_1node.sh
 
# --- 2node-RJ45 ---
cat > /home4cluster/lora_train/run_r${R}_2node_rj45.sh << EOF
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
        -e LORA_R=${R} \\
        nvcr.io/nvidia/pytorch:25.05-py3 \\
        bash -c 'pip install -q peft transformers datasets && torchrun \\
            --nnodes=\$NNODES --nproc_per_node=1 \\
            --master_addr=\$MASTER_ADDR --master_port=\$MASTER_PORT \\
            --node_rank=\$RANK \\
            /home4cluster/lora_train/train_alpaca_lora_rank.py'" &
done; wait
EOF
chmod +x /home4cluster/lora_train/run_r${R}_2node_rj45.sh
 
# --- 2node-QSFP ---
cat > /home4cluster/lora_train/run_r${R}_2node_qsfp.sh << EOF
#!/bin/bash
MASTER_ADDR="10.0.1.1"; MASTER_PORT="29500"; NNODES=2
NODES=("node15" "node16")
for i in "\${!NODES[@]}"; do
    NODE=\${NODES[\$i]}; RANK=\$i
    ssh mprg@\$NODE "docker run --rm --gpus all --network host \\
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \\
        -v /home4cluster:/home4cluster \\
        -v /home/mprg/nccl-build:/home/mprg/nccl-build \\
        -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \\
        -e NCCL_SOCKET_IFNAME=enp1s0f0np0 \\
        -e NCCL_IB_DISABLE=0 \\
        -e NCCL_IB_HCA=rocep1s0f0 \\
        -e CONNECT_TYPE=qsfp \\
        -e LORA_R=${R} \\
        nvcr.io/nvidia/pytorch:25.05-py3 \\
        bash -c 'pip install -q peft transformers datasets && torchrun \\
            --nnodes=\$NNODES --nproc_per_node=1 \\
            --master_addr=\$MASTER_ADDR --master_port=\$MASTER_PORT \\
            --node_rank=\$RANK \\
            /home4cluster/lora_train/train_alpaca_lora_rank.py'" &
done; wait
EOF
chmod +x /home4cluster/lora_train/run_r${R}_2node_qsfp.sh
 
# --- 4node-RJ45 ---
cat > /home4cluster/lora_train/run_r${R}_4node_rj45.sh << EOF
#!/bin/bash
MASTER_ADDR="10.0.0.15"; MASTER_PORT="29500"; NNODES=4
NODES=("node15" "node16" "node17" "node18")
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
        -e LORA_R=${R} \\
        nvcr.io/nvidia/pytorch:25.05-py3 \\
        bash -c 'pip install -q peft transformers datasets && torchrun \\
            --nnodes=\$NNODES --nproc_per_node=1 \\
            --master_addr=\$MASTER_ADDR --master_port=\$MASTER_PORT \\
            --node_rank=\$RANK \\
            /home4cluster/lora_train/train_alpaca_lora_rank.py'" &
done; wait
EOF
chmod +x /home4cluster/lora_train/run_r${R}_4node_rj45.sh
 
done
echo "全12スクリプト生成完了"
```
 
---
 
## ステップ10.3：実験の実行
 
### r=16
 
```bash
# 1node
# /home4cluster/lora_train/logs/1node_none_r16_20260509_143323_steps.csv
bash /home4cluster/lora_train/run_r16_1node.sh
 
# 2node-RJ45
# /home4cluster/lora_train/logs/2node_rj45_r16_20260509_152723_steps.csv
bash /home4cluster/lora_train/run_r16_2node_rj45.sh
 
# 2node-QSFP
bash /home4cluster/lora_train/run_r16_2node_qsfp.sh
 
# 4node-RJ45
bash /home4cluster/lora_train/run_r16_4node_rj45.sh
```
 
### r=32
 
```bash
# 1node
# /home4cluster/lora_train/logs/1node_none_r32_20260509_144933_steps.csv
bash /home4cluster/lora_train/run_r32_1node.sh

# 2node-RJ45
# /home4cluster/lora_train/logs/2node_rj45_r32_20260509_153421_steps.csv
bash /home4cluster/lora_train/run_r32_2node_rj45.sh

# 2node-QSFP
bash /home4cluster/lora_train/run_r32_2node_qsfp.sh

# 4node-RJ45
bash /home4cluster/lora_train/run_r32_4node_rj45.sh
```
 
### r=64
 
```bash
# 1node
# /home4cluster/lora_train/logs/1node_none_r64_20260509_150103_steps.csv
bash /home4cluster/lora_train/run_r64_1node.sh

# 2node(RJ45)
# /home4cluster/lora_train/logs/2node_rj45_r64_20260509_154147_steps.csv
bash /home4cluster/lora_train/run_r64_2node_rj45.sh


bash /home4cluster/lora_train/run_r64_2node_qsfp.sh
bash /home4cluster/lora_train/run_r64_4node_rj45.sh
```

### r=128

```bash
# node1
# /home4cluster/lora_train/logs/1node_none_r128_20260509_151645_steps.csv
bash /home4cluster/lora_train/run_r128_1node.sh

# /home4cluster/lora_train/logs/2node_rj45_r128_20260509_234114_steps.csv
bash /home4cluster/lora_train/run_r128_2node_rj45.sh

bash /home4cluster/lora_train/run_r128_2node_qsfp.sh
bash /home4cluster/lora_train/run_r128_4node_rj45.sh
```
 
---
 
## 結果の確認
 
全実験完了後、以下でサマリーを表示できます。
 
```bash
# 結果一覧（ランク・構成・スループット）
column -t -s, /home4cluster/lora_train/logs/results_lora_rank.csv
```
 
---
 
## 設計上の注意点
 
- **`lora_alpha = r × 2` に統一**：alpha/r の比（スケーリング係数）を全ランクで一定にすることで、ランク変化の効果だけを比較できます。
- **学習可能パラメータ数の目安**：
  - r=8（既存）：約2.5M（全体の0.033%）
  - r=16：約5.0M（0.066%）
  - r=32：約10.0M（0.131%）
  - r=64：約20.1M（0.263%）
- **QSFPの効果が現れ始めるか**：r=64では勾配量が約8倍になるため、LoRAでもRJ45とQSFPの差が出始める可能性があります。
