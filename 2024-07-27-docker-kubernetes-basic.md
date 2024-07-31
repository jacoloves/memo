# docker-kubernetes-basic

**Date:**  2024/07/27

## TL;DR
Dockerの基本的な使い方を学ぶにはとてもよかった。
特にdocker-compose.ymlの書き方を項目ごとに丁寧に説明してくれているので、これを見ながら実際に書いてみると理解が深まる。
nginxやmysqlなどのコンテナを段階的に立ち上げていき、wordpressを立ち上げるところまでやってみた。
コマンドで書いていき、後半にdocker-compose.ymlを書いていくという流れがよかった。
Kubernetesについても軽く触れらていて、PodやServiceなどの概念を学ぶことができた。
KustomizeやHelmに言及していないところもよく、裾野を広げすぎていないので、初学者にはちょうどよい内容だと思う。
もしもっとKubernetesを学ぶ際には、さらに深く学ぶ必要がある。

## chap3
### docker インストール
済み

## chap4
- バージョン確認
```
$docker version
Client:
 Cloud integration: v1.0.31
 Version:           20.10.23
 API version:       1.41
 Go version:        go1.18.10
 Git commit:        7155243
 Built:             Thu Jan 19 17:35:19 2023
 OS/Arch:           darwin/arm64
 Context:           default
 Experimental:      true

Server: Docker Desktop 4.17.0 (99724)
 Engine:
  Version:          20.10.23
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.18.10
  Git commit:       6051f14
  Built:            Thu Jan 19 17:31:28 2023
  OS/Arch:          linux/arm64
  Experimental:     false
 containerd:
  Version:          1.6.18
  GitCommit:        2456e983eb9e37e47538f59ea18f2043c9a73640
 runc:
  Version:          1.1.4
  GitCommit:        v1.1.4-0-g5fd4c4d
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad
```

- 作成・起動
```
$docker run --name apa000ex1 -d httpd                                                                                                                                                                                                                       [main]
Unable to find image 'httpd:latest' locally
latest: Pulling from library/httpd
262a5f25eec7: Pull complete
bac7bacf0601: Pull complete
4f4fb700ef54: Pull complete
6a762f9da5ef: Pull complete
e6566df07573: Pull complete
2697b59c8baf: Pull complete
Digest: sha256:932ac36fabe1d2103ed3edbe66224ed2afe0041b317bcdb6f5d9be63594f0030
Status: Downloaded newer image for httpd:latest
d31e51938fe65c27b81f476e97af1389390e530cb642a0320e300e5cc42cc019
```

- 一覧表示
```
docker ps
```

全てのコンテナを表示
```
docker ps -a
```

- 止める
```
docker stop apa000ex1
```

- 削除する
```
docker rm apa000ex1
```

### コンテナの通信について
```
-p ホストのポート番号:コンテナのポート番号
```

- 作成する
```
docker run --name apa000ex2 -d -p 8080:80 httpd
```

```
docker ps                                                                                                                                                                                                                                                  [main]
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS         PORTS                       NAMES
fc5bc76ee85e   httpd                  "httpd-foreground"       10 seconds ago   Up 9 seconds   0.0.0.0:8080->80/tcp        apa000ex2
```

- 止める
```
docker stop apa000ex2
```

- 削除
```
docker rm apa000ex2
```

## 色々なコンテナを立てる
- run
```
docker run --name apa000ex3 -d -p 8081:80 httpd
docker run --name apa000ex4 -d -p 8082:80 httpd
docker run --name apa000ex5 -d -p 8083:80 httpd
```

- stop
```
docker stop apa000ex3
docker stop apa000ex4
docker stop apa000ex5
```

- delete
```
docker rm apa000ex3
docker rm apa000ex4
docker rm apa000ex5
```

## nvinx
- run
```
docker run --name nginx000ex6 -d -p 8084:80 nginx
```

- stop
```
docker stop nginx000ex6
```

- rm
```
docker rm nginx000ex6
```

## imageを削除
- image ls
```
docker image ls
```

- image rm
```
docker image rm httpd
```

```
docker image rm nginx mysql
```

## wordpressを立ち上げる
- ネットワークを作成する
```
docker network create wordpress000net1
```

- mysqlコンテナを作成する
```
docker run --name mysql000ex11 -dit --net=wordpress000net1 -e MYSQL_ROOT_PASSWORD=myrootpass -e MYSQL_DATABASE=wordpress000db -e MYSQL_USER=wordpress000kun -e MYSQL_PASSWORD=wkunpass mysql --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --default-authentication-plugin=mysql_native_password
```

- wordpress container create
```
docker run --name wordpress000ex12 -dit --net=wordpress000net1 -p 8085:80 -e WORDPRESS_DB_HOST=mysql000ex11 -e WORDPRESS_DB_NAME=wordpress00db -e WORDPRESS_DB_USER=wordpress000kun -e WORDPEWSS_DB_PASSWORD=wkunpass wordpress
```

- stop
```
docker stop wordpress000ex12
docker stop mysql000ex11
```

- rm
```
docker rm wordpress000ex12
docker rm mysql000ex11
```

- image
```
docker image rm wordpress
docker image rm mysql
```

- delete network
```
docker network rm wordpress000net1
```

## コンテナ作る
- run
```
docker run --name apa000ex19 -d -p 8089:80 httpd
```

- docker cp
```
docker cp index.html apa000ex19:/usr/local/apache2/htdocs/
```

- docker cp docker->local
```
docker cp apa000ex19:/usr/local/apache2/htdocs/index.html ./
```

- 停止削除イメージ削除
```
docker stop apa000ex19
docker rm apa000ex19
docker image rm httpd
```

- run
```
docker run --name apa000ex20 -d -p 8090:80 -v /Users/stanaka/tmp/new_lab/2024/docker-sandbox/handson/app_holder:/usr/local/apache2/htdocs httpd
```

- 停止削除
```
docker stop apa000ex20
docker rm apa000ex20
```

## ボリュームをマウントする
- ボリュームを作成
```
docker volume create apa000vol1
```

- docker run
```
docker run --name apa000ex21 -d -p 8091:80 -v apa000vol1:/usr/local/apache2/htdocs httpd
```

- inspect
```
docker volume inspect apa000vol1
docker container inspect apa000ex21
```

- volume delete
```
docker stop apa000ex21
docker rm apa000ex21
docker volume rm apa000vol1
```

## イメージ化
- run
```
docker run --name apa000ex22 -d -p 8092:80 httpd
```

- commit
```
docker commit apa000ex22 ex22_original1
```

- stop-rm
```
docker stop apa000ex22
docker rm apa000ex22
```

- Dockerfile
```
FROM httpd
COPY index.html /usr/local/apache2/htdocs/
```

- build
```
docker build -t ex22_original2 .
```

- stop-rm
```
docker stop ex22_original2
docker rm ex22_original2
```

## chap7
### docker-composeについて学ぶ
- docker-compose.yml
```yml
version: "3"
services:
  mysql000ex11:
    image: mysql:5.7
    networks:
      - wordpress000net1
    volumes:
      - mysql000vol11:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: myrootpass
      MYSQL_DATABASE: wordpress000db
      MYSQL_USER: wordpress000kun
      MYSQL_PASSWORD: wkunpass
  wordpress000ex12:
    depends_on:
      - mysql000ex11
    image: wordpress
    networks:
      - wordpress000net1
    volumes:
      - wordpress000vol2:/var/www/html
    ports:
      - 8085:80
    restart: always
    environment:
      WORDPRESS_DB_HOST: mysql000ex11
      WORDPRESS_DB_NAME: wordpress000db
      WORDPRESS_DB_USER: wordpress000kun
      WORDPRESS_DB_PASSWORD: wkunpass
networks:
  wordpress000net1:
volumes:
  mysql000vol11:
  wordpress000vol2:
```

- docker-compose up
```
docker-compose -f ./docker-compose.yml up -d
```

## chap8
### Kubernetes
- 管理者
  - kubectl
- マスターノード
  コンテナを管理してる
  - etcd
    コンテナなどの状態を保存している
  - kube-apiserver
    マスターノードとワーカーノードの間で通信を行う
- ワーカーノード
  コンテナを実行している
  - kubelet
    マスターノードからの指示を受けてコンテナを実行する
  - kube-proxy
    ネットワークの設定を行う

- pod
  - コンテナをまとめて管理する
- Service
  - podにアクセスするための仮想IPアドレスを提供する
  - podを束ねる
  - ロードバランサー
- ReplicaSet
  - podの数を管理する
  - Replica
    - podの数
- Deployment
  - ReplicaSetを管理する
  - podの数を管理する

- kubeadm
  - kubernetesを簡単に構築するためのツール

### 実際にyamlファイルを作成してデプロイする
- pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apa000pod
  labels:
    app: apa000kube
spec:
  containers:
    - name: apa000ex91
      image: httpd
      ports:
        - containerPort: 80
```

- deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apa000dep
spec:
  selector:
    matchLabels:
      app: apa000kube
  replicas: 3
  template:
    metadata:
      labels:
        app: apa000kube
    spec:
      containers:
        - name: apa000ex91
          image: httpd
          ports:
            - containerPort: 80

```

- service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: apa000ser
spec:
  type: NodePort
  ports:
    - port: 8099
      targetPort: 80
      protocol: TCP
      nodePort: 30080
  selector:
    app: apa000kube
```

### apply
```
kubectl apply -f apa000dep.yml
```

途中applyでエラーが出た場合
```
minikube stop
minikube start
```

kubectl get pod
```
kubectl get pod -n default                                                    [master]
NAME                                          READY   STATUS    RESTARTS   AGE
apa000dep-58cc68ccdf-42lxv                    1/1     Running   0          12m
apa000dep-58cc68ccdf-cr2nl                    1/1     Running   0          12m
apa000dep-58cc68ccdf-mx6nf                    1/1     Running   0          12m
```

#### apply service
```
kubectl apply -f apa000ser.yml
```

- serviceの確認
```
kubectl get service                                                           [master]
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
apa000ser    NodePort    10.102.41.216   <none>        8099:30080/TCP   8s
```

- ブラウザでアクセス
```
minikube service apa000ser
```

### pod数を変えて再apply
- deployment
```yaml
replicas: 5
```

- apply
```
kubectl apply -f apa000dep.yml
```

### イメージをへんこうして再apply
- deployment
```yaml
image: nginx
```

- apply
```
kubectl apply -f apa000dep.yml
```

- ブラウザでアクセス
```
minikube service apa000ser
```

### podの削除
```
kubectl delete pod apa000dep-6d99655b4f-5kwjt -n default                      [master]
pod "apa000dep-6d99655b4f-5kwjt" deleted
```

### yamlから削除
```
kubectl delete -f apa000dep.yml
```

### サービスの削除
```
kubectl delete -f apa000ser.yml
```
