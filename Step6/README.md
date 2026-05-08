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


## ステップ6.2：Dockerの用意
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

## ステップ6.3：NCCLの準備
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

## ステップ6.4：計算node間でのssh鍵の共有
計算node間でssh鍵を共有します．
node15で以下を実行してください．
```
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

ssh-copy-id mprg@10.0.0.15
ssh-copy-id mprg@10.0.0.16
ssh-copy-id mprg@10.0.0.17
ssh-copy-id mprg@10.0.0.18
```


## ステップ6.5：NCCLの多node通信テスト
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




