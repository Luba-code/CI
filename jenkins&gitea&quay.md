# jenkins&gitea&quay

---

![](https://i.imgur.com/MqqeXBx.jpg)

前篇gitea有創好一個儲存庫了，也git clone了

![](https://i.imgur.com/21xbcRw.jpg)

試寫一個jenkinsfile，進到m1的sre-jenkins目錄進行編寫

```
pipeline {
    agent { label 'master' }
    stages {
        stage('hello') {
            steps {
                sh "echo Hello $whoami"
            }
        }
    }
}
```



寫完之後先別push，回jenkins設定

![](https://i.imgur.com/pItKaUT.jpg)

pipeline名稱和設定

![](https://i.imgur.com/eQ5gxBK.jpg)

![](https://i.imgur.com/bSB3q8Q.jpg)

按第一個Jenkinsfile

![](https://i.imgur.com/d5v1t4D.jpg)

檔案憑證提供者

![](https://i.imgur.com/QXBWNTY.jpg)

設定一分鐘追蹤一次

![](https://i.imgur.com/cOz6NDT.jpg)

pipeline就設定好了，(成公球是灰色的會一直閃，失敗是紅色的)

![](https://i.imgur.com/ZrVJ05D.jpg)

回到m1的sre-jenkins把剛剛的hello推上去

![](https://i.imgur.com/T435M6x.jpg)

看jenkins狀態

有錯誤代表jenkinsfile寫錯了，修正再push回去就可以了

![](https://i.imgur.com/3G8mV6l.jpg)

---

![](https://i.imgur.com/MLAKlKM.jpg)

到adM這台主機當作jenkins的節點，需要java環境所以裝openjdk

![](https://i.imgur.com/F6QfSeS.jpg)

```
sudo apt update
sudo apt install -y openjdk-8-jdk openjdk-8-jre
mkdir jenkins_wk
```

到jenkins新增節點，做podman的pull和push

![](https://i.imgur.com/H08foO3.jpg)

![](https://i.imgur.com/stVCviW.jpg)

![](https://i.imgur.com/bPN4zwU.jpg)

節點主機的IP和申請憑證

![](https://i.imgur.com/OxZW4jb.jpg)

ssh 節點主機的帳密

![](https://i.imgur.com/WIqQi4R.jpg)

設定完，等它跑一下就連結上節點了

![](https://i.imgur.com/JSxvest.jpg)

---

![](https://i.imgur.com/lMICiJR.jpg)

* 利用jenkins幫我們image build 和 pull

回到sre-jenkins，創一個alpine.base資料夾，加入dockerfile

192.168.218.22:8088是內網的quay，前篇有mirror了alpine了利用它再做build

## 去到adm那台做podman login

`podman login --tls-verify=false 192.168.218.22:8088`

```
FROM 192.168.218.22:8088/quay/alpine:3.13.5
RUN \
  apk update && \
  apk upgrade && \
  apk add --no-cache nano sudo wget curl tree elinks bash shadow procps util-linux coreutils binutils findutils grep && \
  wget https://busybox.net/downloads/binaries/1.28.1-defconfig-multiarch/busybox-x86_64 && \
  chmod +x busybox-x86_64 && \
  mv busybox-x86_64 bin/busybox1.28 && \
  mkdir -p /opt/www && echo "let me go" > /opt/www/index.html
CMD ["/bin/bash"]
```

修改之前的jenkinsfile，加入pull 和 build 

```
pipeline {
    agent { label 'podman' }
    environment {
        quay_address = '192.168.218.22:8088'
        image_version = '1.0.0'
    }
    stages {
        stage('Image pull') {
            steps {
                sh "podman pull --tls-verify=false ${env.quay_address}/quay/al>
            }
        }
        stage('Image build') {
            steps {
                sh "podman build -t ${env.quay_address}/quay/alpine.base:${env>
            }
        }
        stage('Hello') {
            steps {
                sh "echo HelloHi"
            }
        }
    }
}
```

設定好了jenkins就會自動去抓gitea的更新來幫我們執行，pipeline要執行的專案

![](https://i.imgur.com/I8nlmwY.jpg)

跑了之後發現了錯誤，這時候就要去看log它報錯在哪，原來是podman pull 不下來，去做更正

![](https://i.imgur.com/IueiMmr.jpg)

到adm這台就會發現apline.base:1.0.0已經幫我們建立好了

![](https://i.imgur.com/XeWEGaq.jpg)

* 整個流程圖

![](https://i.imgur.com/XyJYeCg.jpg)



###### tags: `CI/CD`
