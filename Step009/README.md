# ステップ009：NCCLビルドなどのリセット
QSFPスイッチ関連の設定や，ステップ101で設定したNCCLビルドをリセットします．

## QSFPインターフェースの削除
ここまでに何度もQSFPインターフェースのip固定を行なってきたので，一度全てリセットする．

### node15
まずnode15についてリセットを行ってきます．
以下のコマンドを実行してください．
```
# enp1s0f1np1 の状態確認
ip -br addr | grep enp1s0f1np1

# enp1s0f1np1 のIP削除
sudo ip addr flush dev enp1s0f1np1

# 削除確認
ip -br addr | grep enp1s0f1np1

# enP2p1s0f1np1 のIP削除
sudo ip addr flush dev enP2p1s0f1np1

# 削除確認
ip -br addr show dev enP2p1s0f1np1
```

### node16
以下のコマンドを実行してください．
```
# enp1s0f1np1 のIP削除
sudo ip addr flush dev enp1s0f1np1

# 削除確認
ip -br addr show dev enp1s0f1np1

# enP2p1s0f1np1 のIP削除
sudo ip addr flush dev enP2p1s0f1np1

# 削除確認
ip -br addr show dev enP2p1s0f1np1
```

### node17
以下のコマンドを実行してください．
```
# enp1s0f1np1 のIP削除
sudo ip addr flush dev enp1s0f1np1

# 削除確認
ip -br addr show dev enp1s0f1np1

# enP2p1s0f1np1 のIP削除
sudo ip addr flush dev enP2p1s0f1np1

# 削除確認
ip -br addr show dev enP2p1s0f1np1
```


### node18
以下のコマンドを実行してください．
```
# enp1s0f1np1 のIP削除
sudo ip addr flush dev enp1s0f1np1

# 削除確認
ip -br addr show dev enp1s0f1np1

# enP2p1s0f1np1 のIP削除
sudo ip addr flush dev enP2p1s0f1np1

# 削除確認
ip -br addr show dev enP2p1s0f1np1
```

もしくは以下のコマンドを全ての計算nodeで実行して永続的に削除．
```
sudo rm /etc/netplan/40-cx7.yaml
sudo ip addr flush dev enp1s0f1np1
sudo ip addr flush dev enP2p1s0f1np1
ip -br addr show dev enp1s0f1np1
ip -br addr show dev enP2p1s0f1np1
```

## NCCLビルドのリセット
ステップ101でビルドしたNCCLをリセットします．
全てのnodeで`~/nccl`と`~/nccl-tests`を削除します．

### node15
```
rm -rf ~/nccl ~/nccl-tests
```

### nod16
```
rm -rf ~/nccl ~/nccl-tests
```

### node17
```
rm -rf ~/nccl ~/nccl-tests
```

### node18
```
rm -rf ~/nccl ~/nccl-tests
```


## NCCLの再ビルド
NVIDIAの公式手順に従って，NCCLを再ビルドします．

### node15〜node18
```
git clone -b v2.28.9-1 https://github.com/NVIDIA/nccl.git ~/nccl
cd ~/nccl
git branch --show-current
git log -1 --oneline
```

```
# ビルドを実行
make -j src.build NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121"
```

```
# 結果確認
mprg@spark-fb97:~/nccl$ ls -l ~/nccl/build/lib/libnccl.so*
lrwxrwxrwx 1 mprg mprg       12  5月 18 13:04 /home/mprg/nccl/build/lib/libnccl.so -> libnccl.so.2
lrwxrwxrwx 1 mprg mprg       17  5月 18 13:04 /home/mprg/nccl/build/lib/libnccl.so.2 -> libnccl.so.2.28.9
-rwxrwxr-x 1 mprg mprg 50108144  5月 18 13:04 /home/mprg/nccl/build/lib/libnccl.so.2.28.9
```

```
# 環境変数の設定
export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64:$MPI_HOME/lib:$LD_LIBRARY_PATH"
```

```
# nccl-testのクローン
git clone https://github.com/NVIDIA/nccl-tests.git ~/nccl-tests
```

```
# nccl-testのビルド
cd ~/nccl-tests/
make MPI=1
```


## リンク速度の確認
以下のコマンドでリンク速度を確認する．
```
mprg@spark-fb97:~/nccl-tests$ sudo ethtool enp1s0f1np1 | grep -E "Speed|Link detected"
sudo ethtool enP2p1s0f1np1 | grep -E "Speed|Link detected"
	Speed: 200000Mb/s
	Link detected: yes
	Speed: 200000Mb/s
	Link detected: yes
mprg@spark-fb97:~/nccl-tests$ 
```

