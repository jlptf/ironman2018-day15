## Day 15 - Volume (2)

### 本日共賞

* Persistent Volumes
* Persistent Volume Claims

### 希望你知道
* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### Persistent Volumes (PV)

k8s 利用 PV 將提供儲存區 (Storage) 與使用儲存區抽象化。PV 是一個在 k8s 中提供一串儲存區的虛擬化物件，它的概念有點像是 k8s 叢集中的 Node。而在 PV 中可能包含多種 Volume Type 像是 [Day 14 - Volume (1)](https://ithelp.ithome.com.tw/articles/10193546) 中提過的 `gcePersistentDisk ` 或 `awsElasticBlockStore ` 等等。

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062tKXu8nEtd8.png)

<br/>

#### Persistent Volume Claims (PVC)

有了 PV 之後，我們可以透過 PVC 物件取得向 PV 發出要求儲存區的請求。它的概念就像是 Pod 會消耗 Node 的資源 (CPU, Memory) 一般，PVC 可以向 PV 要求存取 (大小、read/write 多次或一次等等) 儲存區。

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062Z127GvCSce.png)

如上圖，PV 管理者根據需求配置 Volume 後 [1]，使用者可以透過 PVC 的方式[2] 取得 PV 的資源進行操作 [3]，最後在掛載到 Pod 使用 [4]。

照理說，需要 Volume 直接掛上去開始使用不就結束了嗎？如果是以前單體式 (monolithic) 的應用的確是直接掛上去最方便使用。但是如果從管理叢集的角度來看，資源的設定與配置應該是屬於管理者的工作。

作為一個稱職的管理者，應該要知道如何選擇適合系統運作的資源 (大小、IOPS等等)。

> 如果想要什麼資源就給什麼資源，那還需要管理叢集嗎？

而做為一個使用者，當需要使用資源時，應該先詢問系統內是否有需要的資源，若有則使用，若系統無法提供需要的資源時，需要考慮是否該向管理者提出增加資源的請求，亦或者檢討需要的資源是否合理。

底下我們來實際操作一下 PV 與 PVC

```
# pv.yaml

---
apiVersion: v1
kind: PersistentVolume    <=== 指定物件種類為 PV
metadata:
  name: pv001		      <=== PV 名稱
spec:
  capacity:
    storage: 2Gi          <=== 指定大小
  accessModes:
  - ReadWriteOnce         <=== 指定存取模式
  hostPath:               <=== 綁定在 host 的 /tmp 目錄
    path: /tmp
```

存取模式 (acessModes) 有三種

1. `ReadWriteOnce `：被單一 Node 掛載為 read/write 模式
2. `ReadOnlyMany `：被多個 Node 掛載為 read 模式
3. `ReadWriteMany `：被多個 Node 掛載為 read/write 模式

> 請注意，存取模式還會受到儲存區的限制。例如：GCE 不支援 Disk 被多個 Node 掛載為 read/write 模式，這時候就需要另外處理。

部署到 k8s

```bash
$ kubectl apply -f pv.yaml
persistentvolume "pv001" created
```

查看一下 PV 的狀態

```bash
$ kubectl get persistentvolumes
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv001     2Gi        RWO            Retain           Available                                      52s
```

這裡 PV 會有四種狀態 (STATUS)

1. `Available`：表示 PV 為可用狀態
2. `Bound`：表示已綁定到 PVC
3. `Released`：PVC 已被刪除，但是尚未回收
4. `Failed`：回收失敗

PV 有三種回收策略 (RECLAIM POLICY)，分別是

1. `Retain`：手動回收
2. `Recycle`：透過刪除命令 `rm -rf /thevolume/*`
3. `Delete`：用於 AWS EBS, GCE PD, Azure Disk 等儲存後端，刪除 PV 的同時也會一併刪除後端儲存磁碟。

了解了 PV 之後，我們再來看看 PVC

```
# pvc.yaml

---
apiVersion: v1
kind: Pod         <=== 使用一個 Pod 並試著掛載 Volume
metadata:
  name: pvc-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:       <=== 將名為 volume-pv 的 Volume 掛載到 /usr/share/nginx/html 目錄底下
      - name: volume-pv
        mountPath: /usr/share/nginx/html
  volumes:
  - name: volume-pv   <=== 宣告一個名為 volume-pv 的 Volume 物件
    persistentVolumeClaim:   <=== 綁定名為 pv-claim 的 PVC 物件
      claimName: pv-claim

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi   <=== 要求 1G 容量
```

為了試驗掛載 PVC，我們宣告了一個名為 `pvc-nginx` 的 Pod 物件。部署 pvc.yaml 到 k8s 中，並查看狀態

```bash
$ kubectl apply -f pvc.yaml
pod "pvc-nginx" created
persistentvolumeclaim "pv-claim" created

$ kubectl get persistentvolumes
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM              STORAGECLASS   REASON    AGE
pv001                                      2Gi        RWO            Retain           Available                                               22m
pvc-9870e19d-dbe7-11e7-9e94-0800279a8379   1Gi        RWO            Delete           Bound       default/pv-claim   standard                 1m

$ kubectl get persistentvolumeclaims
NAME       STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pv-claim   Bound     pvc-9870e19d-dbe7-11e7-9e94-0800279a8379   1Gi        RWO            standard       30s
```

這時候你會看到多了一個狀態為 `Bound` 的 PV，表示已順利取得 PV 資源，可以開始使用了。


更詳細的內容，可以參考 [官網](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)


本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day15/](https://jlptf.github.io/ironman2018-day15/)

