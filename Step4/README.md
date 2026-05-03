## ステップ４：QSFPの2台接続
QSFPケーブル経由でDGX Sparkの2台を接続できるようにします。
まず、node15で以下のコマンドを実行してください。
```
mprg@spark-fb97:~$ ibdev2netdev
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Down)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Up)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Down)
mprg@spark-fb97:~$ 
```
