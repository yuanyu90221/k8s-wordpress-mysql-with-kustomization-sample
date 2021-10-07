# k8s-wordpress-mysql-with-kustomization-sample
## 前言

參考文件 https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/

今天將會透過 kustomize 來佈署 WordPress 以及 MySQL

由於這邊是使用 local 的 minikube

所以對於 PersistentVolumeCliams 將會使用 hostPath 這種 StorageClass

## 佈署目標

1 建立 PersistentVolumClaims 與 PersistentVolume

2 建立 kustomization.yaml 裏面設定下面幾個資源：
   
* SecretGenerator
* MySQL resource configs
* WordPress resource configs

3 利用 kustomization.yaml 佈署整個環境

4 清除佈署


## 建立 PersistentVolumClaims 與 PersistentVolume

MySQL 跟 WordPress 都需要 PersistentVolume 來儲存資料

而這兩個的 PersistentVolumeClaim 都會在各自的 Deployment 檔案來宣告

關於 PersistentVolume 這邊是透過 StorageClass 來處理

如同之前章節 [day12 Persisting Data in K8s with Volumes](https://ithelp.ithome.com.tw/articles/10272861) 所描述

StorageClass 就是由 k8s Administrator 預先定義好的 Storage 配置方式, 當 Deployment 發出 PersistentVolumeClaim, k8s cluster 就會自動配置 PersistentVolume 給 PersistentVolumeClaim 元件讓 Deployment 使用

而在 local cluster 預設是使用 hostPath 的 StorageClass

使用 hostPath 這種 StorageClass, k8s cluster 會把資料存放在 Pod 所在結點的 /tmp 資料夾, 而當 Pod 死掉在其他結點啟動或是結點重啟都會讓資料消失

因此 hostPath 只適合用來測試環境

## 建立 kustomization.yaml

### 建立 kustomization.yaml

首先設定 secretGenerator 來建立 Secret

這個 Secret 會用在 MySQL 跟 WordPress 的 Deployment 內 如下：
```yaml=
secretGenerator:
- name: mysql-pass
  literals:
  - password=mypassword
```

### 建立 mysql-deployment.yaml 如下：

```yaml=
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

設定一個內部 Service 給 mysql-wordpress

設定 PersistentVolumeClaim 給名稱為 mysql-pv-claim 並且設定容量為 20Gi

建立一個 Depylotment

名稱設定為 mysql-wordpress

設定 Deployment 使用 mysql-pv-claim 並且掛載在 /var/lib/mysql

設定 Deployment 使用 image: mysql:5.6

設定 Deployment 從 Secret mysql-pass 帶入 password 值到環境變數 MYSQL_ROOT_PASSWORD


### 建立 wordpress-deployment.yaml 如下：
```yaml=
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
```

設定一個外部 Service 給 wordpress 

設定 PersistentVolumeClaim 給名稱為 wp-pv-claim 並且設定容量為 20Gi

建立一個 Depylotment

名稱設定為 wordpress

設定 Deployment 使用 wp-pv-claim 並且掛載在 /var/www/html

設定 Deployment 使用 image: wordpress:4.8-apache

設定 Deployment 從 Secret mysql-pass 帶入 password 值到環境變數 WORDPRESS_DB_PASSWORD

設定 Deployment Container 環境變數 WORDPRESS_DB_HOST = wordpress-mysql

### 新增上面的兩個 resources 到 kustomization.yaml

新增  mysql-deployment.yaml 與 wordpress-deployment.yaml 到 kustomization.yaml 如下

```yaml=
secretGenerator:
- name: mysql-pass
  literals:
  - password=mypassword
resources:
  - mysql-deployment.yaml
  - wordpress-deployment.yaml
```

## 利用 kustomization.yaml 佈署整個環境

在與 kustomization.yaml 同一個資料夾下 

利用以下指令做佈署：

```shell=
kubectl apply -k ./
```

![](https://i.imgur.com/ymT67Tv.png)


透過以下指令驗證 Secret 佈署

```shell=
kubectl get secrets
```

![](https://i.imgur.com/7JLLFkB.png)

透過以下指令驗證 PersistentVolumeCliam 佈署

```shell=
kubectl get pvc
```

![](https://i.imgur.com/9GsUltm.png)

透過以下指令驗證 Pod 佈署

```shell=
kubectl get pod
```

![](https://i.imgur.com/18f172j.png)


透過以下指令驗證 wordpress 對外的 Service 佈署

```shell=
kubectl get services wordpress
```
![](https://i.imgur.com/gYwWOo8.png)

因為是 minikube 所以需要透過以下指令把 wordpress Service attach 到一個對外 IP

```shell=
minikube service wordpress --url
```
![](https://i.imgur.com/noO3zm2.png)

透過這個對外 IP 我們就可以使用瀏覽器連線到 WordPress 了

![](https://i.imgur.com/kCxpkoI.png)

## 清除佈署

透過 kustomization.yaml 可以使用以下指令清除所有佈署

```yaml=
kubectl delete -k ./
```

![](https://i.imgur.com/nCG6V1q.png)
