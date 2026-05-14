# ステップ101：Dockerなどの環境構築
ここでは，LLMの学習と推論に向けてDockerなどの環境構築を行います．

## ステップ101.1：Dockerグループの追加
DGX SparkのOS再インストール後，Dockerのグループから外れてしまったようなので，グループへの追加からしていく必要がありそうです．
まず，以下のコマンドで所属しているグループを確認してみます．
```
groups mprg
```
出力に`docker`が含まれていれば問題ありません．
`docker`が含まれていなければ，権限がないためDockerの使用ができないです．
以下のコマンドで権限を付与してください．
```
sudo usermod -aG docker mprg
```
これでDockerグループへの追加が完了です．
同様の手順で全ての計算nodeと管理者nodeでDockerグループに追加してください．


## ステップ101.2：Dockerの用意
まず，Dockerの用意から行います．
以下のコマンドを全ての計算nodeで実行してください．
```
cat > /home4cluster/docker/Dockerfile << 'EOF'
FROM nvcr.io/nvidia/pytorch:25.03-py3

# NCCLをBlackwell(sm_100)対応でソースビルド
RUN git clone https://github.com/NVIDIA/nccl.git /tmp/nccl && \
    cd /tmp/nccl && \
    make -j$(nproc) \
      NVCC_GENCODE="-gencode=arch=compute_100,code=sm_100" \
      PREFIX=/usr/local && \
    make install PREFIX=/usr/local && \
    rm -rf /tmp/nccl

ENV LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
EOF
```
```
for node in node15 node16 node17 node18; do
    ssh mprg@$node "docker build -t pytorch-nccl-sm100:latest /home4cluster/docker/" &
done
wait
echo "全nodeのビルド完了"
```

## ステップ101.3：NCCLの準備
全ての計算nodeでNCCLをビルドします．
以下のコマンドを全ての計算nodeで実行してください．
```
cd ~ && git clone https://github.com/NVIDIA/nccl.git
cd nccl
make -j$(nproc) \
  NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121" \
  PREFIX=/home/mprg/nccl-build
make install PREFIX=/home/mprg/nccl-build
```
実行が完了したら，OpenMPIをインストールします．
以下のコマンドを全ての計算nodeで実行してください．
```
sudo apt-get install -y libopenmpi-dev
```
その後，nccl-testのビルドをします．
以下のコマンドを全ての計算nodeで実行してください．
```
cd ~ && git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests
make MPI=1 \
  NCCL_HOME=/home/mprg/nccl-build \
  MPI_HOME=/usr/lib/aarch64-linux-gnu/openmpi
```

## ステップ101.4：計算node間でのssh鍵の共有
計算node間でssh鍵を共有します．
node15で以下を実行してください．
```
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

ssh-copy-id mprg@10.0.0.15
ssh-copy-id mprg@10.0.0.16
ssh-copy-id mprg@10.0.0.17
ssh-copy-id mprg@10.0.0.18
```


## ステップ101.5：NCCLの多node通信テスト
### 2台-RJ45接続
node15で以下のコマンドを実行し，node15とnode16でNCCLテストを行います．
```
export NCCL_IB_DISABLE=1
export NCCL_NET=Socket

mpirun -np 2 \
  -H 10.0.0.15:1,10.0.0.16:1 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME \
  -x UCX_NET_DEVICES \
  -x NCCL_IB_DISABLE \
  -x NCCL_NET \
  /home/mprg/nccl-tests/build/all_reduce_perf -b 8 -e 256M -f 2 -g 1
```

### 2台-QSFP
```
export NCCL_IB_HCA=rocep1s0f0

mpirun -np 2 \
  -H 10.0.1.1:1,10.0.1.2:1 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME \
  -x UCX_NET_DEVICES \
  -x NCCL_IB_DISABLE \
  -x NCCL_NET \
  -x NCCL_IB_HCA \
  /home/mprg/nccl-tests/build/all_reduce_perf -b 8 -e 256M -f 2 -g 1
```

### 4台-RJ45
```
mkdir -p ~/nccl-test-scripts/4node_rj45

cat > ~/nccl-test-scripts/4node_rj45/run.sh << 'EOF'
#!/bin/bash

export NCCL_HOME=/home/mprg/nccl-build
export MPI_HOME=/usr/lib/aarch64-linux-gnu/openmpi
export LD_LIBRARY_PATH=$NCCL_HOME/lib:$MPI_HOME/lib:$LD_LIBRARY_PATH
export NCCL_SOCKET_IFNAME=enP7s7
export UCX_NET_DEVICES=enP7s7
export OMPI_MCA_btl_tcp_if_include=enP7s7
export NCCL_IB_DISABLE=1
export NCCL_NET=Socket

mpirun -np 4 \
  -H 10.0.0.15:1,10.0.0.16:1,10.0.0.17:1,10.0.0.18:1 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME \
  -x UCX_NET_DEVICES \
  -x NCCL_IB_DISABLE \
  -x NCCL_NET \
  /home/mprg/nccl-tests/build/all_reduce_perf -b 8 -e 256M -f 2 -g 1
EOF

chmod +x ~/nccl-test-scripts/4node_rj45/run.sh
```

### 4台-QSFP
スイッチないと無理．
```
```


## ステップ6.6：torchrunによるPyTorchの分散学習テスト
全ての計算nodeで以下のコマンドを実行してください．
```
docker pull nvcr.io/nvidia/pytorch:25.05-py3
```
ホストでビルドしたNCCLライブラリ（sm_121対応、shared memory制限を考慮したもの）をコンテナにマウントして使います．
spark15で以下のコマンドを実行して動作を確認します．
```
docker run --rm --gpus all --network host \
  --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home/mprg/nccl-build:/home/mprg/nccl-build \
  -e LD_PRELOAD=/home/mprg/nccl-build/lib/libnccl.so \
  nvcr.io/nvidia/pytorch:25.05-py3 \
  python3 -c "
import torch
import torch.distributed as dist
import os

os.environ['MASTER_ADDR'] = 'localhost'
os.environ['MASTER_PORT'] = '29500'
os.environ['RANK'] = '0'
os.environ['WORLD_SIZE'] = '1'

dist.init_process_group(backend='nccl')
tensor = torch.ones(1).cuda()
dist.all_reduce(tensor)
print(f'single node NCCL test OK: {tensor.item()}')
dist.destroy_process_group()
"
```


### 2node（RJ45）-torchrun + PyTorch分散通信テスト
管理者nodeで以下のコマンドを実行してください．
```
mkdir -p /home4cluster/torchrun_test

cat > /home4cluster/torchrun_test/nccl_test.py << 'EOF'
import torch
import torch.distributed as dist
import os

def main():
    dist.init_process_group(backend="nccl")
    rank = dist.get_rank()
    world_size = dist.get_world_size()

    torch.cuda.set_device(0)
    tensor = torch.ones(1).cuda() * rank
    print(f"[rank {rank}/{world_size}] before all_reduce: {tensor.item()}")

    dist.all_reduce(tensor, op=dist.ReduceOp.SUM)
    print(f"[rank {rank}/{world_size}] after all_reduce: {tensor.item()}")

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
EOF
```
続いて管理者nodeで起動スクリプトを作成します．
以下のコマンドを実行してください．
```
cat > /home4cluster/torchrun_test/run_2node_rj45.sh << 'EOF'
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
        -e NCCL_DEBUG=WARN \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        torchrun \
        --nnodes=$NNODES \
        --nproc_per_node=1 \
        --master_addr=$MASTER_ADDR \
        --master_port=$MASTER_PORT \
        --node_rank=$RANK \
        /home4cluster/torchrun_test/nccl_test.py" &
done
wait
EOF
chmod +x /home4cluster/torchrun_test/run_2node_rj45.sh
```
その後，管理者nodeで以下のコマンドを実行してください．
```
bash /home4cluster/torchrun_test/run_2node_rj45.sh
```


### 2node（QSFP）-torchrun + PyTorch分散通信テスト
管理者nodeで起動スクリプトを作成します．
```
cat > /home4cluster/torchrun_test/run_2node_qsfp.sh << 'EOF'
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
        -e NCCL_DEBUG=WARN \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        torchrun \
        --nnodes=$NNODES \
        --nproc_per_node=1 \
        --master_addr=$MASTER_ADDR \
        --master_port=$MASTER_PORT \
        --node_rank=$RANK \
        /home4cluster/torchrun_test/nccl_test.py" &
done
wait
EOF
chmod +x /home4cluster/torchrun_test/run_2node_qsfp.sh
```
実行します．
```
bash /home4cluster/torchrun_test/run_2node_qsfp.sh
```


### 4node（RJ45）-torchrun + PyTorch分散通信テスト
管理者nodeで起動スクリプトを作成します．
```
cat > /home4cluster/torchrun_test/run_4node_rj45.sh << 'EOF'
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
        -e NCCL_DEBUG=WARN \
        nvcr.io/nvidia/pytorch:25.05-py3 \
        torchrun \
        --nnodes=$NNODES \
        --nproc_per_node=1 \
        --master_addr=$MASTER_ADDR \
        --master_port=$MASTER_PORT \
        --node_rank=$RANK \
        /home4cluster/torchrun_test/nccl_test.py" &
done
wait
EOF
chmod +x /home4cluster/torchrun_test/run_4node_rj45.sh
```
実行します．
```
bash /home4cluster/torchrun_test/run_4node_rj45.sh
```




