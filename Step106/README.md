# ステップ106：LLM学習速度調査（QSFPも含めて再計測）
QSFPスイッチも含めて再計測を行います．
ステップ6〜8でQSFP直結の設定を削除してしまっているので，それらの復元などもここで行います．

## LoRAチューニング学習速度比較結果
### 実験設定

| 項目 | 値 |
|------|-----|
| モデル | Qwen2.5-7B-Instruct |
| データセット | tatsu-lab/alpaca（1,000件） |
| シーケンス長 | 512トークン |
| バッチサイズ | 1 |
| LoRA rank (r) | 8 |
| LoRA alpha | 16 |
| 対象モジュール | q_proj, v_proj |
| 学習可能パラメータ | 2,523,136（全体の0.033%） |
| ウォームアップステップ | 5 |
| seed | 42 |
| フレームワーク | PyTorch DDP + PEFT |
| Dockerイメージ | nvcr.io/nvidia/pytorch:25.05-py3 |

### 結果

|     構成    |    throughput    |    elapsed    | avg_step_time  | speedup | scaling efficiency |
|------------|------------------|---------------|----------------|---------|--------------------|
| 1node      |  2.14 samples/sec |    465.4s     |     0.450s     |  1.00x  | 100%               |
| 2node-RJ45 |  4.17 samples/sec |    237.2s     |     0.461s     |  x  | %                |
| 2node-QSFP |  4.20 samples/sec |    235.9s     |     0.460s     |  x  | %                |
| 4node-RJ45 |  8.28 samples/sec |    118.3s     |     0.465s     |  x  | %                |
| 4node-QSFP |  8.29 samples/sec |    118.2s     |     0.466s     |  x  | %                |


### 考察


## フルFT学習速度比較結果
### 実験設定

| 項目 | 値 |
|------|-----|
| モデル | Qwen2.5-7B-Instruct |
| データセット | tatsu-lab/alpaca（1,000件） |
| シーケンス長 | 512トークン |
| バッチサイズ | 1 |
| LoRA rank (r) | 8 |
| LoRA alpha | 16 |
| 対象モジュール | q_proj, v_proj |
| 学習可能パラメータ | 7,615,616,512（全体の100%） |
| ウォームアップステップ | 5 |
| seed | 42 |
| フレームワーク | PyTorch DDP + PEFT |
| Dockerイメージ | nvcr.io/nvidia/pytorch:25.05-py3 |

### 結果

|     構成    |    throughput    |    elapsed    | avg_step_time  | speedup | scaling efficiency |
|------------|------------------|---------------|----------------|---------|--------------------|
| 1node      |  2.14 samples/sec |    465.4s     |     0.450s     |  1.00x  | 100%               |
| 2node-RJ45 |  4.17 samples/sec |    237.2s     |     0.461s     |  x  | %                |
| 2node-QSFP |  4.20 samples/sec |    235.9s     |     0.460s     |  x  | %                |
| 4node-RJ45 |  8.28 samples/sec |    118.3s     |     0.465s     |  x  | %                |
| 4node-QSFP |  8.29 samples/sec |    118.2s     |     0.466s     |  x  | %                |




## 前準備
### `40-cx7.yaml`の作成
QSFPケーブルでDGX Sparkを直接接続するために，`40-cx7.yaml`を再作成します．
各計算nodeで以下のコマンドを実行してください．
```
# node15
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.1.1/24
      dhcp4: no
EOF
sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
ip a show enp1s0f0np0 | grep inet
```

```
# node16
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.1.2/24
      dhcp4: no
EOF
sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
ip a show enp1s0f0np0 | grep inet
```

```
# node17
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.2.1/24
      dhcp4: no
EOF
sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
ip a show enp1s0f0np0 | grep inet
```

```
# node18
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.2.2/24
      dhcp4: no
EOF
sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
ip a show enp1s0f0np0 | grep inet
```

### `/etc/hosts`への追記
全てのnodeで以下のコマンドを実行してください．
```
sudo bash -c 'cat >> /etc/hosts << EOF

# QSFP direct connection (10.0.1.x / 10.0.2.x)
10.0.1.1   node15-qsfp
10.0.1.2   node16-qsfp
10.0.2.1   node17-qsfp
10.0.2.2   node18-qsfp
EOF'
```

### SSH鍵配布（node15で実行）
現在QSFPスイッチへと接続しているので，できていません．
後でやります．


## ステップ106.1：環境確認
全ての計算nodeで以下を実行して環境を確認します．
```
# NCCLの確認
ls -l ~/nccl/build/lib/libnccl.so*

# 3系統のネットワーク確認
ip a show enP7s7       | grep inet  # RJ45
ip a show enp1s0f0np0  | grep inet  # QSFP直結
ip a show enp1s0f1np1  | grep inet  # QSFPスイッチ
```


## ステップ106.2：学習スクリプトの用意
学習プログラムは，すでに作成済みの`/home4cluster/lora_train/train_alpaca.py`を使用します．
起動スクリプトは全て`/home4cluster/lora_train/step106/`ディレクトリに配置します．
以下のコマンドを管理者nodeで実行して起動スクリプトを作成してください．
```
# LoRA用スクリプト
# ── 1node ──────────────────────────────────────────────────────
cat > /home4cluster/lora_train/step106/run_lora_1node.sh << 'EOF'
#!/bin/bash
ssh mprg@node15 "docker run --rm --gpus all --network host \
    --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
    -v /home4cluster:/home4cluster \
    -v /home/mprg/nccl/build:/home/mprg/nccl/build \
    -e LD_PRELOAD=/home/mprg/nccl/build/lib/libnccl.so \
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

# ── 2node RJ45 ─────────────────────────────────────────────────
cat > /home4cluster/lora_train/step106/run_lora_2node_rj45.sh << 'EOF'
#!/bin/bash
MASTER_ADDR="10.0.0.15"; MASTER_PORT="29500"; NNODES=2
NODES=("node15" "node16")
for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}; RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl/build:/home/mprg/nccl/build \
        -e LD_PRELOAD=/home/mprg/nccl/build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enP7s7 \
        -e NCCL_IB_DISABLE=1 \
        -e NCCL_NET=Socket \
        -e CONNECT_TYPE=rj45 \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers datasets && torchrun \
            --nnodes=$NNODES --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train_alpaca.py'" &
done; wait
EOF

# ── 2node QSFP直結 ─────────────────────────────────────────────
cat > /home4cluster/lora_train/step106/run_lora_2node_qsfp.sh << 'EOF'
#!/bin/bash
MASTER_ADDR="10.0.1.1"; MASTER_PORT="29500"; NNODES=2
NODES=("node15-qsfp" "node16-qsfp")
for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}; RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl/build:/home/mprg/nccl/build \
        -e LD_PRELOAD=/home/mprg/nccl/build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enp1s0f0np0 \
        -e NCCL_IB_DISABLE=0 \
        -e NCCL_IB_HCA=rocep1s0f0 \
        -e CONNECT_TYPE=qsfp \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers datasets && torchrun \
            --nnodes=$NNODES --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train_alpaca.py'" &
done; wait
EOF

# ── 2node QSFPスイッチ ─────────────────────────────────────────
cat > /home4cluster/lora_train/step106/run_lora_2node_qsfp_sw.sh << 'EOF'
#!/bin/bash
MASTER_ADDR="192.168.100.15"; MASTER_PORT="29500"; NNODES=2
NODES=("node15-sw" "node16-sw")
for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}; RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl/build:/home/mprg/nccl/build \
        -e LD_PRELOAD=/home/mprg/nccl/build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enp1s0f1np1 \
        -e NCCL_IB_DISABLE=0 \
        -e NCCL_IB_HCA=rocep1s0f1 \
        -e CONNECT_TYPE=qsfp_sw \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers datasets && torchrun \
            --nnodes=$NNODES --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train_alpaca.py'" &
done; wait
EOF

# ── 4node RJ45 ─────────────────────────────────────────────────
cat > /home4cluster/lora_train/step106/run_lora_4node_rj45.sh << 'EOF'
#!/bin/bash
MASTER_ADDR="10.0.0.15"; MASTER_PORT="29500"; NNODES=4
NODES=("node15" "node16" "node17" "node18")
for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}; RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl/build:/home/mprg/nccl/build \
        -e LD_PRELOAD=/home/mprg/nccl/build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enP7s7 \
        -e NCCL_IB_DISABLE=1 \
        -e NCCL_NET=Socket \
        -e CONNECT_TYPE=rj45 \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers datasets && torchrun \
            --nnodes=$NNODES --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train_alpaca.py'" &
done; wait
EOF

# ── 4node QSFPスイッチ ─────────────────────────────────────────
cat > /home4cluster/lora_train/step106/run_lora_4node_qsfp_sw.sh << 'EOF'
#!/bin/bash
MASTER_ADDR="192.168.100.15"; MASTER_PORT="29500"; NNODES=4
NODES=("node15-sw" "node16-sw" "node17-sw" "node18-sw")
for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}; RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -v /home/mprg/nccl/build:/home/mprg/nccl/build \
        -e LD_PRELOAD=/home/mprg/nccl/build/lib/libnccl.so \
        -e NCCL_SOCKET_IFNAME=enp1s0f1np1 \
        -e NCCL_IB_DISABLE=0 \
        -e NCCL_IB_HCA=rocep1s0f1 \
        -e CONNECT_TYPE=qsfp_sw \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        bash -c 'pip install -q peft transformers datasets && torchrun \
            --nnodes=$NNODES --nproc_per_node=1 \
            --master_addr=$MASTER_ADDR --master_port=$MASTER_PORT \
            --node_rank=$RANK \
            /home4cluster/lora_train/train_alpaca.py'" &
done; wait
EOF

chmod +x /home4cluster/lora_train/step106/run_lora_*.sh
echo "LoRA起動スクリプト生成完了"
```

```
# フルFT用スクリプト
for conf in 1node 2node_rj45 2node_qsfp 2node_qsfp_sw 4node_rj45 4node_qsfp_sw; do
    sed "s/train_alpaca\.py/train_alpaca_full.py/g; s/run_lora/run_full/g" \
        /home4cluster/lora_train/step106/run_lora_${conf}.sh \
        > /home4cluster/lora_train/step106/run_full_${conf}.sh
    chmod +x /home4cluster/lora_train/step106/run_full_${conf}.sh
done
echo "フルFT起動スクリプト生成完了"
```

## ステップ106.3：LoRA学習の実行

### 1node
```
bash /home4cluster/lora_train/step106/run_lora_1node.sh
```
```
=== 結果 ===
steps        : 1000 (warmup: 5, measured: 995)
avg_loss     : 0.2934
elapsed      : 465.4s
throughput   : 2.14 samples/sec
avg_step_time: 0.450s/step

ステップログ : /home4cluster/lora_train/logs/1node_none_20260519_060443_steps.csv
サマリーログ : /home4cluster/lora_train/logs/results.csv
```

### 2node（RJ45）
```
bash /home4cluster/lora_train/step106/run_lora_2node_rj45.sh
```
```
=== 結果 ===
steps        : 500 (warmup: 5, measured: 495)
avg_loss     : 0.3132
elapsed      : 237.2s
throughput   : 4.17 samples/sec
avg_step_time: 0.461s/step

ステップログ : /home4cluster/lora_train/logs/2node_rj45_20260519_061942_steps.csv
サマリーログ : /home4cluster/lora_train/logs/results.csv
```


### 4node（RJ45）
```
bash /home4cluster/lora_train/step106/run_lora_4node_rj45.sh
```
```
=== 結果 ===
steps        : 250 (warmup: 5, measured: 245)
avg_loss     : 0.3864
elapsed      : 118.3s
throughput   : 8.28 samples/sec
avg_step_time: 0.465s/step

ステップログ : /home4cluster/lora_train/logs/4node_rj45_20260519_065029_steps.csv
サマリーログ : /home4cluster/lora_train/logs/results.csv
```


### 2node（QSFPスイッチ）
```
bash /home4cluster/lora_train/step106/run_lora_2node_qsfp_sw.sh
```
```
=== 結果 ===
steps        : 500 (warmup: 5, measured: 495)
avg_loss     : 0.3172
elapsed      : 235.9s
throughput   : 4.20 samples/sec
avg_step_time: 0.460s/step

ステップログ : /home4cluster/lora_train/logs/2node_qsfp_sw_20260519_064236_steps.csv
サマリーログ : /home4cluster/lora_train/logs/results.csv
```


### 4node（QSFPスイッチ）
```
bash /home4cluster/lora_train/step106/run_lora_4node_qsfp_sw.sh
```
```
=== 結果 ===
steps        : 250 (warmup: 5, measured: 245)
avg_loss     : 0.3820
elapsed      : 118.2s
throughput   : 8.29 samples/sec
avg_step_time: 0.466s/step

ステップログ : /home4cluster/lora_train/logs/4node_qsfp_sw_20260519_065807_steps.csv
サマリーログ : /home4cluster/lora_train/logs/results.csv
```


