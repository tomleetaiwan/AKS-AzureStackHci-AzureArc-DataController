
# 之前準備
請先依據文件 https://docs.microsoft.com/en-us/azure/azure-arc/data/install-client-tools 安裝妥所需的命令列工具，並依照 https://docs.microsoft.com/en-us/azure-stack/aks-hci/create-kubernetes-cluster 內容以 Windows Admin Center 建立妥 AKS on Azure Stack HCI 

在 Windows Admin Center 中取得所建立 Kubernetes 叢集名稱。![從 Windows Admin Center 內取得 Kubernetes 叢集名稱](/images/cluster-name.png)

進入 Azure Stack Windows Server Host 環境，啟動 PowerShell 並鍵入以下命令 
```ps
Get-AksHciCredential -Name <叢集名稱> -Confirm:$false
```
完成之後可以透過 kubectl 列出所有 Kubernetes 節點確認已經可以順利連線。

```ps
kubectl get node
```

# 在 AKS on Azure Stack HCI 上建立 Linux Ext4 之 StorageClass 

鍵入以下命令建立儲存體容器，以這個範例會建立一個名為 customStorageContainer 的儲存體容器在 V:\AksHCI\customStorage 路徑。

```ps
New-AksHciStorageContainer -Name customStorageContainer -Path V:\AksHCI\customStorage
```
完成後建入以下命令確認儲存體容器已經順利建立完畢‧
```ps
Get-AksHciStorageContainer -Name customStorageContainer
```
鍵入以下命令顯示目前預設 StorageClass 完整的 YAML 定義
```code
kubectl get storageclass default -o yaml
```
顯示結果會類似下面畫面，請紀錄下這個 YAML 檔案中的 group 與 hostname 內容。

![紀錄 group 與 hostname 名稱](/images/default-storage-class-yaml.png)

[下載 Linux Ext4 StorageClass 的 YAML 範本](./custom-storage.yaml)並儲存為一個名為 custom-storage.yaml 的檔案，開啟編輯器將此範本中的 group 與 hostname 替換成之前紀錄的內容。

![以編輯器替換範本中 group 與 hostname 名稱](/images/ext4-storage-class-yaml.png)

完成修改儲存內容，並鍵入以下命令註冊 Linux Ext4 StorageClass 至 Kubernetes 叢集。
```code
kubectl apply -f custom-storage.yaml
```
# 以 Linux Ext4 之 StorageClass 建立間接連線 (Indirectly connected) 模式之 Azure Arc Data Controller
透過以下命令以 Azure CLI  在 ./custom 資料夾建立 Azure Arc Data Controller 組態設定檔 control.json
```azurecli
az arcdata dc config init --source azure-arc-aks-hci --path ./custom 
```
透過 kubectl 取得目前所有可用之 StorageClass 
```code
kubectl get storageclass
```
如下畫面顯示結果，請記錄下之前建立之 Linux Ext4 StorageClass 名稱，以畫面中的範例是 Ext4 StorageClass 名稱是 aks-hci-disk-custom
![顯示所有可用之 StorageClass 名稱](/images/ext4-storage-class-name.png)

執行以下命令將 Azure Arc Data Controller 組態設定檔 control.json 內與資料儲存相關的組態設定取代為 Ext4 的 StorageClass 名稱 aks-hci-disk-custom。
```azurecli
az arcdata dc config replace --path ./custom/control.json --json-values "spec.storage.data.className=aks-hci-disk-custom"
```
執行以下命令將 Azure Arc Data Controller 組態設定檔 control.json 內與日誌儲存相關的組態設定取代為 Ext4 的 StorageClass 名稱 aks-hci-disk-custom。
```azurecli
az arcdata dc config replace --path ./custom/control.json --json-values "spec.storage.logs.className=aks-hci-disk-custom"
```
建入以下命令，開始在 AKS on Azure Stack HCI 上以 azure-arc-data 這個 Kubernetes Namespace 建立使用 Linux Ext4 Storage Class 之 Azure Arc Data Controller 
```azurecli
az arcdata dc create --path ./custom --k8s-namespace azure-arc-data --use-k8s --name arc --subscription <訂閱帳號 ID> --resource-group <資源群組名稱> --location <選定之資料中心> --connectivity-mode indirect
```
開始建立之前會要求給定 Azure Arc Data Controller 所需之帳號與密碼，建立時間需數分鐘，待建立完畢之後，透過以下命令確認 Azure Arc Data Controller 相關 Pod 已經在 Namespace azure-arc-data 之內順利運行。

```code
kubectl get pod -n azure-arc-data
```
![建立 Azure Arc Data Controller](/images/azure-arc-data-controller-pods.png)
