# はじめての Azure Kubernetes Service(AKS)
AKSを始めるための環境セットアップ/デプロイ/動作確認

## MacOSのセットアップ
MacOSをAKSのクライアントにするための環境インストール

### Azure CLI
```
brew install azure-client
az login
```

### リソースプロバイダ有効
いらないかもだけど？？
```
az provider register -n Microsoft.Network
az provider register -n Microsoft.Storage
az provider register -n Microsoft.Compute
az provider register -n Microsoft.ContainerService
```

### Kubectl
```
brew install kubernetes-cli
```


## Azure Container Registry
Azure Container Registry は、プライベート Docker コンテナー イメージを格納するために使用される、Azure ベースの管理されたコンテナー レジストリ サービスです。

### レジストリ作成のための環境変数
```
ACR_NAME=acr0225
ACR_RES_GROUP=yama-acr
```
値はユニークなものを設定ください

### リソースグループの作成
```
az group create --resource-group $ACR_RES_GROUP --location japaneast
```

### ACRのレジストリ作成
```
az acr create --resource-group $ACR_RES_GROUP --name $ACR_NAME --sku Standard --location japaneast
```
コマンド出力で、loginServerが、"acr0225.azurecr.io"（指定したACR名）になっていることを確認

### テスト用サンプルコードのダウンロード
```
git clone https://github.com/nowkoai/aks.git
cd aks/dockerImage
```

### イメージのビルド
★az acr buildコマンドでイメージをビルドして、ACRにプッシュします。
```
az acr build --registry $ACR_NAME --image photo-view:v1.0 v1.0/
az acr build --registry $ACR_NAME --image photo-view:v2.0 v2.0/
```
とりあえず、２つのバージョンをACRにプッシュ（あとで、バージョンアップしてみる）

### イメージの確認
```
az acr repository show-tags -n $ACR_NAME --repository photo-view
```
ブラウザ/AzurePortalで、コンテナレジストリの確認してみる！


## ACRとAKSの連携のためのID/パスワード
### ACRのリソースID
```
az acr show --name $ACR_NAME --query id --output tsv
ACR_ID=$(az acr show --name $ACR_NAME --query id --output tsv)
```

### サービスプリンシパル名の環境変数
```
SP_NAME=sample-acr-service-principal
```

### サービスプリンシパルのパスワード！
```
az ad sp create-for-rbac --name $SP_NAME --role Reader --scopes $ACR_ID --query password --output tsv
SP_PASSWD=$(az ad sp create-for-rbac --name $SP_NAME --role Reader --scopes $ACR_ID --query password --output tsv)
```

### サービスプリンシパルのID！
```
az ad sp show --id http://$SP_NAME --query appId --output tsv
APP_ID=$(az ad sp show --id http://$SP_NAME --query appId --output tsv)
```


## AKSでの、Kubernetesクラスターの構築
### クラスタの作成のための環境変数
```
AKS_CLUSTER_NAME=AKSCluster
AKS_RES_GROUP=yama_akscluster
```

### リソースグループの作成
```
az group create --resource-group $AKS_RES_GROUP --location japaneast
```


### AKSでKubernetesクラスタを作成
```
az aks create \
  --name $AKS_CLUSTER_NAME \
  --resource-group $AKS_RES_GROUP \
  --node-count 3 \
  --kubernetes-version 1.19.7 \
  --node-vm-size Standard_DS2_v2 \
  --generate-ssh-keys \
  --service-principal $APP_ID \
  --client-secret $SP_PASSWD
```
ACRとAKS連携のためのパラーメータあるよね


### Kubernetesクラスタ接続のための認証情報の取得
```
az aks get-credentials --admin --resource-group $AKS_RES_GROUP --name $AKS_CLUSTER_NAME
```
.kube/configファイルに接続情報を書き込むことで、MacOSからクラスタ操作できるようになる！


## kubectlコマンドを使ったクラスターの操作
------------------------------------
※bash_profileとかに、export |grep KUBECONFIGがあったので削除
※kubectl config view で、接続先確認できる！

==コマンド例==
kubectl get pod
kubectl get deployment
kubectl get horizontalpodautoscalers

==コマンド実践==
クラスターの情報確認（K8sで動いてるAPIやアドオン機能状態がわかる）
kubectl cluster-info

K8sクラスター上で動くNodeの一覧
kubectl get node
--> ３台のNodeが動いている！

＃-o=wideをつけると、Nodeの追加情報表示（IPやOSバージョン等）
kubectl get node -o=wide

＃３台のNodeのうち、1台の詳細確認
kubectl describe node aks-nodepool1-25600466-vmss000000

＃クラスタノードから、1台目の名前のみ表示
kubectl get node -o=jsonpath='{.items[0].metadata.name}'

＃ヘルプ
kubectl help

※kubectl補完
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc


------------------------------------
★課金が気になる場合の対応
------------------------------------
ACR_NAME=acr0225
ACR_RES_GROUP=yama-acr
az group delete -name $ACR_RES_GROUP

AKS_CLUSTER_NAME=AKSCluster
AKS_RES_GROUP=yama_akscluster
az group delete -name $yama_akscluster

SP_NAME=sample-acr-service-principal
az ad sp delete --id=$(az ad sp show --id http://$SP_NAME --query appId --output tsv)


------------------------------------
■コンテナ/アプリケーションのデプロイの流れ
------------------------------------
#1. マニフェストファイルの作成: ymlファイル
・クラスターにどのようなアプリをデプロイして、クライアントからのアクセスをどう処理するかなどの構成情報を定義ファイルで管理
・CPU/Memory/NetworkAddressなどのリソース
--> deployment.yaml
--> service.yaml

#2. クラスタでのリソース作成: kubectlコマンド ※ACRからイメージのpull
・kubectlコマンドの引数に、マニフェストファイルを指定
（デプロイしたコンテナアプリケーションやネットワーク構成などをリソースと呼ぶ）

・Kubernetesクラスタによって、アプリケーションを適切な場所に配置
--> kubectl apply -f deployment.yaml
・クラスター外部からアクセスするためのネットワークを構成
--> kubectl apply -f service.yaml

#3. アプリケーションの動作確認: http/httpsアクセス
・デプロしされたアプリケーションをクラスタ外のネットワークからアクセスして動作確認
・PCブラウザでアプリが表示できたら確認完了

------------------------------------
ACR（コンテナイメージ） ==> AKS（Kubernetes）--Node（仮想マシン）--コンテナー
--開発者: kubectlコマンド
--ユーザー: アプリケーションの確認（http/https）
------------------------------------
deployment.yaml
service.yaml
------------------------------------

==※事前確認==
--Node状態
＃AKSでは、NodeはAzure仮想マシン（Standard_DS1_v2がワーカーノードとして３つ起動してクラスタを構成）
kubectl get node
kubectl get node -o wide
--> これらのサーバにアプリケーションをデプロイしていく

--Pod状態
＃Pod: コンテナアプリケーションの集合体
kubectl get pod
--> アプリはまだデプロイされていない


==★アプリケーションのデプロイ（Podを適切な場所に配置）==
＃マニフェストをクラスターに送る
kubectl apply -f deployment.yaml
kubectl get pod
kubectl get pod  -o wide
--> ５つのPodがRunning稼働 on クラスタNode:3台

==★サービスの公開（外部アクセスのためのネットワーク構成）==
＃クラスターにデプロイしたアプリケーションにアクセスするためのIPアドレスを確認
kubectl apply -f service.yaml
kubectl get svc

＃ロードバランサー構成: リクエストが各Podに分散されてることを確認
http://Exteral-IP

==デプロイしたPodの詳細確認==
kubectl describe pods photoview-deployment-58669896bd-7nhcf

------------------------------------
★クラスタ上のアプリとネットワークの削除コマンド
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml

--> Kubernetesクラスタを構成する３つのAzure仮想マシンは動いたまま
------------------------------------
