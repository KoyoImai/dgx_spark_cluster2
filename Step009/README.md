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

### node15
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

