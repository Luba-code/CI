# jenkins佈署

---

官方網站

https://www.jenkins.io/

GitHub

https://github.com/jenkinsci/jenkins

---

* 佈署jenkins，跟postgresql和quay一樣

![](https://i.imgur.com/rYbjB6w.jpg)

```
sudo mkdir -p /opt/pv/jenkins-home
sudo setfacl -m u:1000:-wx /opt/pv/jenkins-home/
sudo getfacl /opt/pv/jenkins-home
```

設定jenkins跑在工作機

```
export K8S_EXTERNAL_IP="192.168.218.22"
kubectl apply -f ns.yml
kubectl apply -f pod-jenkins.yml
cat svc-jenkins.yml | envsubst | kubectl apply -f -
```

jenkins的yaml檔案，版本和工作機注意

```
apiVersion: v1
kind: Pod
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins
spec:
  containers:
    - name: jenkins
      image: quay.io/flysangel/jenkins:2.303.2-alpine
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 8080
      volumeMounts:
      - name: jenkins-home
        mountPath: /var/jenkins_home
  volumes:
    - name: jenkins-home
      hostPath:
        path: /opt/pv/jenkins-home
        type: DirectoryOrCreate
  nodeSelector:
    kubernetes.io/hostname: w1-ubuntu
  restartPolicy: Always
```

jenkins的svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: jenkins
  labels:
    app: jenkins
spec:
  externalIPs:
    - ${K8S_EXTERNAL_IP}
  ports:
    - name: jenkins-http
      port: 8888
      targetPort: 8080
  selector:
    app: jenkins
```

namespace

```
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
```

## 登入jenkins

打開瀏覽器進入http://192.168.218.22:8888

![](https://i.imgur.com/9BiqgTH.jpg)

進入jenkins需要初始密碼，可以打下面那串得到

`kubectl logs jenkins -n jenkins`

![](https://i.imgur.com/KeCC0qi.jpg)

進入後需要下載jenkins的元件，等它下載，或是去w1的/opt/pv/jenkins-home，把元件資料放在那，再`sudo chown -R 1000:1000 /opt/pv/jenkins-home/plugins`

![](https://i.imgur.com/9uwVav1.jpg)

元件載完了之後，會要你創造管理帳號，都以root來創建

![](https://i.imgur.com/i72jFfm.jpg)

![](https://i.imgur.com/kn64nTM.jpg)

完成後進入，右上角警告看一下，紅色就要注意，我的是舊版本有些元件有漏洞。

![](https://i.imgur.com/s4WyTua.jpg)


###### tags: `CI/CD`
