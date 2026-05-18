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

## node16
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

## node17
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


## node18
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



