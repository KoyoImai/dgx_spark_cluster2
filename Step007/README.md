# ステップ007：QSFPの4台接続（QSFPスイッチ）

QSFPケーブルでDGX Sparkを4台接続します．
接続にはQSFPスイッチを使用します．
構成は以下の通りです．
```
node15(enp1s0f0np1) ↔ QSFPスイッチ（ケーブル1）
node16(enp1s0f0np1) ↔ QSFPスイッチ（ケーブル2）
node17(enp1s0f0np1) ↔ QSFPスイッチ（ケーブル3）
node18(enp1s0f0np1) ↔ QSFPスイッチ（ケーブル4）
```


## Phase1：既存設定のリセット
ステップ6ではうまく接続できなかったため，設定をリセットします，

