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


## ステップ6.2：Dockerイメージのpull
まず，Dockerイメージのpullから行います．
以下のコマンドを全ての計算nodeで実行してください．
```
docker pull nvcr.io/nvidia/pytorch:25.03-py3
```



