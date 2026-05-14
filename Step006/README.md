# ステップ006：QSFPの4台接続（QSFPスイッチ）

QSFPケーブルでDGX Sparkを4台接続します．
接続にはQSFPスイッチを使用します．
構成は以下の通りです．
```
node15(enp1s0f0np0) ↔ QSFPスイッチ（ケーブル1）
node16(enp1s0f0np0) ↔ QSFPスイッチ（ケーブル2）
node17(enp1s0f0np0) ↔ QSFPスイッチ（ケーブル3）
node18(enp1s0f0np0) ↔ QSFPスイッチ（ケーブル4）
```
（追記5/3 16:19）
以下の構成でipアドレスを固定しました。
```
【既存ペア】
node15(enp1s0f0np0): 10.0.1.1 ↔ node16(enp1s0f0np0): 10.0.1.2
node17(enp1s0f0np0): 10.0.2.1 ↔ node18(enp1s0f0np0): 10.0.2.2

【追加クロス】
node15(enp1s0f1np1): 10.0.3.1 ↔ node17(enp1s0f1np1): 10.0.3.2
node16(enp1s0f1np1): 10.0.4.1 ↔ node18(enp1s0f1np1): 10.0.4.2
```
