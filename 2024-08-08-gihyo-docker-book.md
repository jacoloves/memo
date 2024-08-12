# Docker/Kubernetes 実践コンテナ開発入門を読んだ

**Date:** 2024-08-08 08:13:54

## chap1
- Dockerの基礎的な話が多かった
- オーケストレーションツールとしてDocker Swarmが出したが流行らず、Kubernetesが流行った
- ローリングアップデート
  - バージョンアップデートを行う際に、一度に全てのコンテナをアップデートするのではなく、少しずつアップデートしていく方法

## chap2
- コンテナのデプロイについて
- イメージのタグを省略するとlatestが使われる
- コマンド一覧
```
docker image pull ubuntu:23.10
docker container run ubuntu:23.10 uname -a
docker container run --name echo -it -p 9000:8080 ghcr.io/gihyodocker/echo:v0.1.0
docker container stop echo
```

main.goとDockerfileを作成して、コンテナをビルドして実行する
main.go
```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		log.Println("Received request")
		fmt.Fprintf(w, "Hello Container!!")
	})

	log.Println("Start server")
	server := &http.Server{Addr: ":8081"}

	go func() {
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("ListenAndServe(): %s", err)
		}
	}()

	guit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Println("Shutting down server...")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(ctx); err != nil {
		log.Fatalf("Shutdown(): %s", err)
	}

	log.Println("Server terminated")
}
```

Dockerfile
```
FROM golang:1.20.5

WORKDIR /go/src/github.com/gihyodocker/echo
COPY main.go .
RUN go mod init

CMD ["go", "run", "main.go"]
```

コンテナのビルドと実行
```
docker image build --no-cache -t ch02/echo:latest .
```

コンテナ内に入る
```
docker conainer run -it golang:1.20.5
```

コンテナポートをポートフォワーディングしないと通信できない
```
(つ・ω・)つdocker container run -d -p 9000:8081 ch02/echo:latest                                                             [master]
e32db7f4a703d4dacc357fb6ab3a70fb8f161327bd364141ec3b1f82152ffee8
(base) /Users/stanaka/tmp/new_lab/2024/gihyo-docker-book/ch02/echo 24-08-09 6:54:17
 (つ・ω・)つcurl http://localhost:9000                                                                                        [master]
Hello Container!!%
```

- docker imageで-fオプションを使うと、Dockerfileを指定してビルドできる
```
docker image build -f Dockerfile.dev -t ch02/echo:latest .
```

- 新しいタグをつける
```
docker image tag ch02/echo:latest ch02/echo:0.0.1
```

- filterオプションでイメージを絞り込む
```
docker container ls --filter "name=echo"                                                                          [master]
CONTAINER ID   IMAGE              COMMAND            CREATED         STATUS         PORTS                     NAMES
00e0155a457f   ch02/echo:latest   "go run main.go"   2 minutes ago   Up 2 minutes   0.0.0.0:51757->8081/tcp   echo2
1c4fefc312b0   ch02/echo:latest   "go run main.go"   2 minutes ago   Up 2 minutes   0.0.0.0:51756->8081/tcp   echo1
```

- イメージの検索はfilter ancestorで行う
```
CONTAINER ID   IMAGE              COMMAND            CREATED         STATUS         PORTS                     NAMES
00e0155a457f   ch02/echo:latest   "go run main.go"   5 minutes ago   Up 5 minutes   0.0.0.0:51757->8081/tcp   echo2
1c4fefc312b0   ch02/echo:latest   "go run main.go"   5 minutes ago   Up 5 minutes   0.0.0.0:51756->8081/tcp   echo1
34d6bb21a4bb   ch02/echo:latest   "go run main.go"   22 hours ago    Up 22 hours    0.0.0.0:56869->8081/tcp   frosty_hopper
e32db7f4a703   ch02/echo:latest   "go run main.go"   22 hours ago    Up 22 hours    0.0.0.0:9000->8081/tcp    sad_kalam
```

- pruneでコンテナを削除する
```
docker container prune
```

- pruneでイメージを削除する
```
docker image prune
```

- docker system pruneで全て削除する
```
docker system prune
```

## chap3
- Docker Composeについて
- Data volumeコンテナについて学んだ
```
version: "3.9"

services:
  mysql:
    container_name: mysql
    image: mysql:8.0.33
    environment:
      MYSQL_ALLOW_EMPTY_PASSWORD: yes
      MYSQL_DATABASE: volume_test
    volumes_from:
      - mysql_data

mysql_data:
  container_name: mysql-data
  build: .
```
## chap4
- 実用的なアプリケーションのコンテナ構成について学んだ
- コンテナオーケストレーションの初歩的な概念について学ぶため6台のコンテナを構築した
- Tiltがコンテナ管理に便利
```
https://tilt.dev/
```

## chap5
- kubernetesの基本的な概念について学ぶ
- デバッグPodを作成する
```
kubectl run -i --tty --rm debug --image=ubuntu:20.04 --restart=Never -- bash -il
```

- pod,service,replicaset,deployment,ingressのyamlを書いた
https://github.com/jacoloves/new_lab/commit/5dad89c4bf1fe6b19c5b013b3a1e8c7a86120466

## chap6
- 実際にタスクアプリのインフラを構築していく
- namespaceを作成する
```
kubectl create namespace taskapp                                                                                                                                                           [master]
namespace/taskapp created
```

- kubectlの-oオプションでyamlを出力する
```
(つ・ω・)つkubectl get secret/test-secret -o yaml                                                                                                                                                     [master]
apiVersion: v1
data:
 password: Z2loeW9fcGFzc3dvcmQ=
kind: Secret
metadata:
 creationTimestamp: "2024-08-10T06:59:15Z"
 name: test-secret
 namespace: default
 resourceVersion: "489279"
 uid: 280c24c0-601f-46c2-8580-2c3a58027cd4
type: Opaqu
```

- yamlをたくさん書いて。無事アプリの疎通確認できた。
```
https://github.com/jacoloves/new_lab/commit/3c5d7db485df1123f787fa24dc84d5ba986df7f0
```

- minikubeを使用してる場合、localhostでアクセスできないので、minikube tunnelを使った
  - minikube tunnelを使うと、minikubeのVMにルーティングルールを追加して、localhostからminikubeのサービスにアクセスできるようになる
- minikube ipでminikubeのIPアドレスを取得して、アクセスする方法もあるらしい
  - その場合、マニフェストファイルのserviceのtypeをNodePortに変更する必要がある
```
minikube tunnel
```

- AKSへのデプロイはスキップした

- 掃除が面倒くさかった。。。
- ツールでも作ろうかな　
```
10896  kubectl -n taskapp delete -f web.yaml
10897  kubectl -n taskapp delete -f mysql
10898  kubectl -n taskapp delete -f mysql.yaml
10899  kubectl -n taskapp delete -f mysql-secret.yaml
10900  kubectl -n taskapp delete -f migrator.yaml
10901  kubectl -n taskapp delete -f api.yaml
10902  kubectl -n taskapp delete -f api-config-secret.yaml
```

## chap7
- kubenetesのpod戦略について学ぶ
- live,readinessProbeについて学ぶ
  - liveProbeはコンテナが生きているかどうかを確認する
  - readinessProbeはコンテナがリクエストを受け付ける準備ができているかどうかを確認する
- Graceful Shutdownについて学ぶ
  -　SIGTERMを受け取ったら、リクエストを受け付けないようにする
  - リクエストを受け付けないようにするために、readinessProbeを使う
- RBACについて学ぶ
  - Role,RoleBinding,ClusterRole,ClusterRoleBindingについて学ぶ
  - Roleはnamespace内のリソースに対する権限を設定する
  - ClusterRoleはクラスタ全体のリソースに対する権限を設定する
  - RoleBindingはRoleをユーザーに割り当てる
  - ClusterRoleBindingはClusterRoleをユーザーに割り当てる
- 書いたyaml
```
https://github.com/jacoloves/new_lab/commit/61bb7533594eb2ebb61220ee66bd2669103bf0d0
```

## chap8
- KustomizeとHelmについて学ぶ
- Kustomizeは開発環境や本番環境など複数の環境にデプロイする際に、必要最小限のマニフェストファイルの管理をするために利用する
- Helmは、Kustomizeより細かいマニフェストファイルの制御がひつような場合に利用する

- kustomize create --autodetectでkustomization.yamlを作成する
```
(つ・ω・)つkustomize create --autodetect                                                                                                                                                                                                                              [master]
(base) /Users/stanaka/tmp/new_lab/2024/gihyo-docker-book/k8s/kustomize/echo/base 24-08-11 14:15:27
(つ・ω・)つbat kustomization.yaml                                                                                                                                                                                                                                     [master]
───────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
      │ File: kustomization.yaml
───────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  1   │ apiVersion: kustomize.config.k8s.io/v1beta1
  2   │ kind: Kustomization
  3   │ resources:
  4   │ - deployment.yaml
  5   │ - ingress.yaml
  6   │ - service.yaml
───────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

- kustomize buildでマニフェストファイルを生成する
```
(つ・ω・)つkustomize build .                                                                                                                                                                                                                                          [master]
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: echo
  name: echo
spec:
  ports:
  - name: echo
    port: 80
    protocol: TCP
    targetPort: http
  selector:
    app.kubernetes.io/name: echo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: echo
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: echo
  template:
    metadata:
      labels:
        app.kubetnetes.io/name: echo
    spec:
      containers:
      - env:
        - name: NGINX_PORT
          value: "80"
        - name: SERVER_NAME
          value: localhost
        - name: BECKEND_HOST
          value: localhost:8080
        - name: BACKEND_MAX_FAILS
          value: "3"
        - name: BACKEND_FAILE_TIMEOUT
          value: 10s
        image: ghcr.io/gihyodocker/simple-nginx-proxy:v0.1.0
        name: nginx
        ports:
        - containerPort: 80
          name: http
      - image: ghcr.io/gihyodocker/echo:v0.1.0
        name: echo
---
apiVersion: networking.k8s.io/v1
kind: Ingerss
metadata:
  labels:
    app.kubernetes.io/name: echo
  name: echo
spec:
  ingeressClassName: nginx
  rules:
  - host: echo.gihyo.local
    http:
      paths:
      - backend:
          service:
            name: echo
            port:
              number: 80
        path: /
        pathType: Prefix
```

- helmについて
- bitnamiが一番使われている

- helm installのコマンド
```
(つ・ω・)つhelm install mysql bitnami/mysql \                                                                                                                                                                         [master]
--namespace helm-mysql \
--create-namespace \
--version 9.12.5 \
--values mysql.yaml
NAME: mysql
LAST DEPLOYED: Sun Aug 11 15:34:26 2024
NAMESPACE: helm-mysql
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 9.12.5
APP VERSION: 8.0.34

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace helm-mysql

Services:

  echo Primary: mysql.helm-mysql.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace helm-mysql mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.34-debian-11-r75 --namespace helm-mysql --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mysql.helm-mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

- helm upgradeのコマンド
```
(つ・ω・)つhelm upgrade mysql bitnami/mysql --version 9.12.5 -f mysql.yaml -n helm-mysql                                                                                                                              [master]
Release "mysql" has been upgraded. Happy Helming!
NAME: mysql
LAST DEPLOYED: Sun Aug 11 15:39:07 2024
NAMESPACE: helm-mysql
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 9.12.5
APP VERSION: 8.0.34

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace helm-mysql

Services:

  echo Primary: mysql.helm-mysql.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace helm-mysql mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.34-debian-11-r75 --namespace helm-mysql --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mysql.helm-mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

- chartのアンインストールはリリース名を指定する
```
(つ・ω・)つhelm delete mysql -n helm-mysql                                                                                                                                                                            [master]
release "mysql" uninstalled
```

- helmを書いたがバージョンのベースが変わっていたため動かなかった

- 書いたyaml
https://github.com/jacoloves/new_lab/commit/348c677ebf689d3fc1bfbcbcfcdddf2a6f7fcea3

## chap9
- 可用性のあるコンテナ運用を学んだ
- HPAを使ってスケールアウトを学んだ
- ロギングとモニタリングを学んだ
  - Kiabanaのコンテナを立ち上げる
  - sternを使ってログを見る
- HPAとは
  - リソースの使用率を監視して、設定した閾値を超えたらPodを増減する
  - Podの数を自動で調整する
  - Podの数を増やすときはDeploymentのreplicasを変更する
  - Podの数を減らすときはPodを削除する
  - Podの数を増やすときはPodを作成する
  - Podの数を減らすときはPodを削除する

- 書いたyaml
https://github.com/jacoloves/new_lab/commit/491c056bc41f6bab1be66882935512ff3f861d59

## chap10
- 最適なコンテナイメージについて
- コンテナの容量が小さい
  - Distroless
  - Scratch
  - Alpine
- latestタグは使わない
  - イメージのバージョンを固定する

- マルチステージビルド
  - ビルドと実行のためのコンテナイメージを分ける
  - ビルド用のコンテナイメージはビルドツールをインストールしてビルドする
  - 実行用のコンテナイメージはビルドした成果物だけをコピーする

- 書いたコード
https://github.com/jacoloves/new_lab/commit/9b6b422cdfe5b406a6b9c3c4301defb3965348fc

## chap11
- KubernetesのCI/CD周りについて
- GitOpsについて
  - GitOpsはKubernetesのリソースをGitリポジトリで管理する
  - Gitリポジトリにリソースの定義を保存して、それをKubernetesに適用する
  - リソースの変更はGitリポジトリにプッシュするだけでKubernetesに適用される
  - リソースの変更履歴がGitリポジトリに残る
  - リソースの変更をレビューすることができる
  - リソースの変更をロールバックすることができる
  - リソースの変更を自動で適用することができる
- 代表的なツール
  - ArgoCD
  - Flux
- 特にArgoCDは仕事でバリバり使ってるので、ハンズオンをやってみた
https://argo-cd.readthedocs.io/en/stable/getting_started/

## chap12
- チームで開発するためのDocker環境の知識
- 書いたコード
https://github.com/jacoloves/new_lab/commit/e35ed2f3068d83cc6cd77a8b16cdfdedf8d8e4f0
