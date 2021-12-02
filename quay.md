# 內網的quay.io佈署

---

紅帽公司商業版 quay.io

https://www.redhat.com/en/technologies/cloud-computing/quay

這次佈署的Project Quay

https://www.projectquay.io/

---

* postgresql佈署

quay.io的資料庫(postgresql)PV

![](https://i.imgur.com/GAOQOye.jpg)

```
sudo mkdir -p /opt/pv/postgresql-quay
sudo setfacl -m u:26:-wx /opt/pv/postgresql-quay
sudo getfacl /opt/pv/postgresql-quay
```

要下載acl，才能用setfacl

```
sudo apt install acl
```

![](https://i.imgur.com/6w3rluH.jpg)

quay的PV也是一樣

![](https://i.imgur.com/QWvpmuK.jpg)

```
sudo mkdir -p /opt/pv/quay
sudo setfacl -m u:1001:-wx /opt/pv/quay
sudo getfacl /opt/pv/quay
```

啟動postgresql的yaml

![](https://i.imgur.com/TpZaFya.jpg)

```
kubectl apply -f ns.yml
kubectl apply -f pod-postgresql.yml
kubectl apply -f svc-postgresql.yml

kubectl exec -n quay -it postgresql -- /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | psql -d quay -U postgres'
```

![](https://i.imgur.com/il6Rg0y.jpg)

postgresql的yaml

hostname和版本代號要注意，quay.io/flysangel/ 是明城老師的小舖

```
apiVersion: v1
kind: Pod
metadata:
  name: postgresql
  namespace: quay
  labels:
    app: postgresql
spec:
  containers:
    - name: postgresql
      image: quay.io/flysangel/rhel8-postgresql-10:1-154
      imagePullPolicy: IfNotPresent
      env:
        - name: POSTGRESQL_USER
          value: quayuser
        - name: POSTGRESQL_PASSWORD
          value: quaypass
        - name: POSTGRESQL_DATABASE
          value: quay
        - name: POSTGRESQL_ADMIN_PASSWORD
          value: adminpass
      volumeMounts:
        - name: postgresql-quay
          mountPath: /var/lib/pgsql/data
  volumes:
    - name: postgresql-quay
      hostPath:
        path: /opt/pv/postgresql-quay
        type: DirectoryOrCreate
  nodeSelector:
    kubernetes.io/hostname: w1-ubuntu
  restartPolicy: Always
```

postgresql的svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: quay
spec:
  clusterIP: None
  selector:
    app: postgresql
```

---

* redis和quay config

quay config產生我們的quay.io的設定檔

![](https://i.imgur.com/IhZi3QR.png)

IP改自己work機的

```
export K8S_EXTERNAL_IP="192.168.218.22"
kubectl apply -f pod-redis.yml
kubectl apply -f svc-redis.yml
cat pod-quay-config.yml | envsubst | kubectl apply -f -
```

redis的yaml

注意版本代號可能會變動

```
apiVersion: v1
kind: Pod
metadata:
  name: redis
  namespace: quay
  labels:
    app: redis
spec:
  containers:
    - name: redis
      image: quay.io/flysangel/rhel8-redis-5:1-141
      imagePullPolicy: IfNotPresent
      env:
        - name: REDIS_PASSWORD
          value: strongpassword
  restartPolicy: Always
```

redis的svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: quay
spec:
  clusterIP: None
  selector:
    app: redis
```

quay config的yaml

```
apiVersion: v1
kind: Service
metadata:
  name: quay-config-service
  namespace: quay
  labels:
    app: quay-config
spec:
  externalIPs:
    - ${K8S_EXTERNAL_IP}
  ports:
    - name: quay-config-http
      port: 8081
      targetPort: 8080
  selector:
    app: quay-config
---
apiVersion: v1
kind: Pod
metadata:
  name: quay-config
  namespace: quay
  labels:
    app: quay-config
spec:
  containers:
    - name: quay-config
      image: quay.io/flysangel/rhel8-quay:v3.5.1
      imagePullPolicy: IfNotPresent
      command: ["dumb-init", "--"]
      args: ["/quay-registry/quay-entrypoint.sh", "config", "secret"]
      ports:
        - containerPort: 8080
  restartPolicy: Always
```

postgresql，redis，quay-config都佈署好了

![](https://i.imgur.com/8zyTIuH.jpg)
  
* 打開瀏覽器進入http://192.168.218.22:8081開始設定quay的設定檔

打開會先要你登入，輸入帳號:quayconfig密碼:secret

![](https://i.imgur.com/Jdu7VwJ.jpg)

設定quay待會佈署的位置，port號8088

![](https://i.imgur.com/EcV403l.jpg)

k8s內部的DNS幫我們postgresql的名稱解析

```
postgresql.quay.svc.cluster.local:5432
quayuser
quaypass
quay
```

![](https://i.imgur.com/EHLnYl6.jpg)

```
redis.quay.svc.cluster.local
strongpassword
```

![](https://i.imgur.com/upEnFPj.jpg)

![](https://i.imgur.com/HdVSgP7.jpg)

![](https://i.imgur.com/xF5keYC.jpg)

以上都設定好了，點選左下角的Validate Configuration Changes，下載設定檔案，假如失敗的話代表前面設定有誤

![](https://i.imgur.com/OY0jmTP.jpg)

下載好後解壓縮會出現config壓模檔

![](https://i.imgur.com/79Z4eN0.jpg)

拖進quay目錄底下

![](https://i.imgur.com/kLW8EwL.jpg)

---

## 佈署內網的quay

```
kubectl -n quay create configmap quay-config --from-file quay
kubectl apply -f pod-quay.yml
kubectl apply -f pod-mirroring-worker.yml
cat svc-quay.yml | envsubst | kubectl apply -f -
```

要記得要有export K8S_EXTERNAL_IP="192.168.218.22"，前面沒漏掉這邊就沒問題

完成之後

![](https://i.imgur.com/kxVXGCl.jpg)

quay的yaml(pod位置要改，版本也要注意)

```
apiVersion: v1
kind: Pod
metadata:
  name: quay
  namespace: quay
  labels:
    app: quay
spec:
  initContainers:
  - name: quay-init-config
    image: quay.io/flysangel/busybox:1.33.1
    command: ['sh', '-c', 'cp /quay-config/* /quay-podwk']
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: quay-config
      mountPath: /quay-config
    - name: quay-podwk
      mountPath: /quay-podwk
  containers:
    - name: quay
      image: quay.io/flysangel/rhel8-quay:v3.5.1
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: quay
          mountPath: /datastorage
        - name: quay-podwk
          mountPath: /conf/stack
  volumes:
    - name: quay-config
      configMap:
        name: quay-config
    - name: quay-podwk
      emptyDir: {}
    - name: quay
      hostPath:
        path: /opt/pv/quay
        type: DirectoryOrCreate
  nodeSelector:
    kubernetes.io/hostname: w1-ubuntu
  restartPolicy: Always
```

quay的svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: quay-service
  namespace: quay
  labels:
    app: quay
spec:
  externalIPs:
    - ${K8S_EXTERNAL_IP}
  ports:
    - name: quay-http
      port: 8088
      targetPort: 8080
  selector:
    app: quay
```
mirroring的yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: mirroring-worker
  namespace: quay
  labels:
    app: mirroring-worker
spec:
  initContainers:
  - name: quay-init-config
    image: quay.io/flysangel/busybox:1.33.1
    command: ['sh', '-c', 'cp /quay-config/* /quay-podwk']
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: quay-config
      mountPath: /quay-config
    - name: quay-podwk
      mountPath: /quay-podwk
  containers:
    - name: mirroring-worker
      image: quay.io/flysangel/rhel8-quay:v3.5.1
      imagePullPolicy: IfNotPresent
      command: ["dumb-init", "--"]
      args: ["/quay-registry/quay-entrypoint.sh", "repomirror"]
      volumeMounts:
        - name: quay-podwk
          mountPath: /conf/stack
  volumes:
    - name: quay-config
      configMap:
        name: quay-config
    - name: quay-podwk
      emptyDir: {}
  restartPolicy: Always
```

* 打開瀏覽器進入http://192.168.218.22:8088

自己創一個使用者用quay當密號的話，系統化判定你為superuser

![](https://i.imgur.com/ZlTkL7B.jpg)

![](https://i.imgur.com/zH1i8gd.jpg)

登入後創建Repository

![](https://i.imgur.com/Uwbgxvv.jpg)

創一個alpine儲存庫

![](https://i.imgur.com/Od42CPi.jpg)

建好之後去此儲存庫的設定，開啟mirror

![](https://i.imgur.com/xprTu3E.jpg)

到mirror的設定

![](https://i.imgur.com/llZjMLH.jpg)

建一個追蹤的robot

![](https://i.imgur.com/Z0kLJJt.jpg)

可以看有沒有儲存拷貝到

![](https://i.imgur.com/RGzuGTC.jpg)

需要用內網的quay的image來做pod的話必須下這些指令，在你K8S環境中的每台

```
sudo sed -i \
's|/usr/bin/crio|/usr/bin/crio --insecure-registry=192.168.218.22:8088 --registry=192.168.218.22:8088|g' \
/lib/systemd/system/crio.service
sudo systemctl daemon-reload
sudo systemctl restart crio
```

![](https://i.imgur.com/zqLDk0d.jpg)


###### tags: `CI/CD`
