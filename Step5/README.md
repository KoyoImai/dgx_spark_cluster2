## ステップ５：QSFPの4台接続（2ペア＋クロス接続）
QSFPケーブルでDGX Sparkを4台接続します。
接続構成は「2ペア＋クロス接続」で、以下の構成にします。
```
【現在】
node15(enp1s0f0np0) ↔ node16(enp1s0f0np0)（ケーブル1・既存）
node17(enp1s0f0np0) ↔ node18(enp1s0f0np0)（ケーブル2・既存）

【追加】
node15(enp1s0f1np1) ↔ node17(enp1s0f1np1)（ケーブル3・新規）
node16(enp1s0f1np1) ↔ node18(enp1s0f1np1)（ケーブル4・新規）
```

### node15でip固定

