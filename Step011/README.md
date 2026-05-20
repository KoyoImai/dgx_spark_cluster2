# ステップ11：QSFPスイッチの速度低下原因調査
QSFPスイッチ接続時に，通信速度（nccl-test）が低下する原因を調査する．

## 



## メモ
### スイッチ接続 & 再起動時
```
# FEC の確認
mprg@spark-fb97:~$ sudo ethtool --show-fec enp1s0f1np1
[sudo] mprg のパスワード: 
FEC parameters for enp1s0f1np1:
Supported/Configured FEC encodings: Auto
Active FEC encoding: RS
mprg@spark-fb97:~$ 
```
```

```




