# gitea佈署

---

Gitea 是一個輕量級程式碼託管解決方案，後端使用 Go 程式語言撰寫，採用 MIT 授權條款。

* gitea官網

https://gitea.io/zh-tw/

* gitea的壓模檔，安裝在k8s裡面

quay.io/flysangel/ 是明城老師的image小舖

最下面的hostname記得改

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea
  namespace: gitea
  labels:
    app: gitea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      containers:
        - name: gitea
          image: quay.io/flysangel/gitea:1.14.2
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: gitea-data
              mountPath: /data
          ports:
            - containerPort: 3000
            - containerPort: 22
      volumes:
        - name: gitea-data
          hostPath:
            path: /opt/pv/gitea-data
            type: DirectoryOrCreate
      nodeSelector:
        kubernetes.io/hostname: w1-ubuntu
```

* gitea的namespace 

```
apiVersion: v1
kind: Namespace
metadata:
  name: gitea
```

* gitea的對外port號，可以在本機瀏覽器上操作

```
apiVersion: v1
kind: Service
metadata:
  name: gitea-service
  namespace: gitea
  labels:
    app: gitea
spec:
  externalIPs:
    - ${K8S_EXTERNAL_IP}
  ports:
    - name: gitea-http
      port: 3000
    - name: gitea-ssh
      port: 30022
      targetPort: 22
  selector:
    app: gitea
```

把三個yaml檔案都建立起來

```
kubectl apply -f ns.yml
kubectl apply -f dep-gitea.yml
export K8S_EXTERNAL_IP="192.168.218.22"
cat svc-gitea.yml | envsubst | kubectl apply -f -
```

192.168.218.22是我的work機

![](https://i.imgur.com/asHuPqe.jpg)

---

* gitea初始化

打開瀏覽器輸入 http://192.168.218.22:3000

![](https://i.imgur.com/I4ZsauO.jpg)

更改主機位置，gitea的URL和新增管理者帳密，好了就按新增安裝gitea

![](https://i.imgur.com/LkyeVor.jpg)

進入gitea可以看到右下有個網站管理，只有系統管理者才能看到

![](https://i.imgur.com/D7BlMwN.jpg)

---

* 新增gitea2的儲存庫

新增儲存庫(右上角+號的下拉選單)

設定儲存庫名稱和初始化

![](https://i.imgur.com/oJzJajZ.jpg)

![](https://i.imgur.com/A0rB0DW.jpg)

設定好之後待會clone的位置

![](https://i.imgur.com/YTBk6Mg.jpg)

回到VScode做git clone

`git clone http://192.168.218.22:3000/root/sre-jenkins.git`

![](https://i.imgur.com/pkL0xqg.jpg)

完成之後就可以push程式碼上去

![](https://i.imgur.com/iapgz5Z.jpg)

###### tags: `CI/CD`
