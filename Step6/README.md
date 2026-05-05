# ステップ６：Dockerなどの環境構築
ここでは，LLMの学習と推論に向けてDockerなどの環境構築を行います．

## ステップ6.1：Dockerグループの追加
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


## ステップ6.2：Dockerイメージのpull
まず，Dockerイメージのpullから行います．
以下のコマンドを全ての計算nodeで実行してください．
```
docker pull nvcr.io/nvidia/pytorch:25.03-py3
```

## ステップ6.3：NCCLの多node通信テスト
NCCLによって多node通信テストを行います．
管理者nodeで以下の内容のファイルを`/home4cluster/nccl_test/nccl_test.py`に作成してください．
```
import torch
import torch.distributed as dist
import os

def main():
    dist.init_process_group(backend="nccl")
    rank = dist.get_rank()
    world_size = dist.get_world_size()
    local_rank = int(os.environ.get("LOCAL_RANK", 0))

    torch.cuda.set_device(local_rank)
    device = torch.device("cuda", local_rank)

    tensor = torch.ones(1).to(device) * rank
    print(f"[rank {rank}/{world_size}] before all_reduce: {tensor.item()}")

    dist.all_reduce(tensor, op=dist.ReduceOp.SUM)
    print(f"[rank {rank}/{world_size}] after all_reduce: {tensor.item()}")

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
```
続いて，起動スクリプトも作成します．
以下の内容のファイルを`/home4cluster/nccl_test/run_nccl_test.sh`に作成してください．
```
#!/bin/bash

MASTER_ADDR="10.0.0.15"
MASTER_PORT="29500"
NNODES=4
NPROC_PER_NODE=1

NODES=("node15" "node16" "node17" "node18")

for i in "${!NODES[@]}"; do
    NODE=${NODES[$i]}
    RANK=$i
    ssh mprg@$NODE "docker run --rm --gpus all --network host \
        --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
        -v /home4cluster:/home4cluster \
        -e NCCL_SOCKET_IFNAME=enP7s7 \
        -e NCCL_IB_DISABLE=0 \
        -e NCCL_DEBUG=INFO \
        nvcr.io/nvidia/pytorch:25.03-py3 \
        torchrun \
        --nnodes=$NNODES \
        --nproc_per_node=$NPROC_PER_NODE \
        --master_addr=$MASTER_ADDR \
        --master_port=$MASTER_PORT \
        --node_rank=$RANK \
        /home4cluster/nccl_test/nccl_test.py" &
done
wait
```




