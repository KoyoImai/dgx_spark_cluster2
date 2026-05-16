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
ステップ5とステップ6ではうまく接続できなかったため，設定をリセットします，
まず，ステップ5で作成した`/etc/netplan/41-cx7-p2.yaml`を削除します．
以下のコマンドを計算nodeで実行してください．
```
sudo rm -f /etc/netplan/41-cx7-p2.yaml
sudo netplan apply
ls /etc/netplan/
sudo sed -i '/qsfp2/d' /etc/hosts
sudo sed -i '/qsfp-sw/d' /etc/hosts
```


## Phase 2：MikroTikスイッチの設定
### MikroTikスイッチ（CRS812-DDQ）の設定手順
QSFPスイッチにLANケーブルを接続し，管理者nodeから一時的にIPアドレスを付与します．
以下のコマンドを実行してください．
```
sudo ip addr add 192.168.88.2/24 dev enx6c6e0705ec11
ping -c 3 192.168.88.1
```

pingが問題なく通っていれば、ブラウザで`http://192.168.88.1`を開いてログインします。
ログイン後以下の手順で設定を進めてください。

#### 1:ブリッジの確認
`≡` → `Bridge` → Bridge タブ を開き、bridge1（デフォルト）が存在するか確認。
なければ作成。




