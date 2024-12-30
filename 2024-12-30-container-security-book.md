# container security book

**Date:** 2024-12-30 08:24:45

https://gihyo.jp/book/2023/978-4-297-13635-2
基礎から学ぶコンテナセキュリティ――Dockerを通して理解するコンテナの攻撃例と対策を読んで感想を書く。

## 1章
コンテナについて基礎知識が書かれている
```
♥ ❯ docker run --rm -it -d ubuntu sleep 10
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
8bb55f067777: Pull complete
Digest: sha256:80dd3c3b9c6cecb9f1667e9290b3bc61b78c2678c02cbdae5f0fea92cc6734ab
Status: Downloaded newer image for ubuntu:latest
31cdc1bbd5d9c132d71c59c98a32f818c52ce0e59ca18f986c466fb874de0f46

♥ ❯ ps aux | grep sleep
stanaka          49367   0.0  0.0 408505024   1264 s004  R+    8:28AM   0:00.00 grep sleep
```
コンテナで実行したプロセスをホストから確認

```
❯ docker run --rm -it ubuntu ps auxf
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1 13.0  0.0   7628  3208 pts/0    Rs+  23:30   0:00 ps auxf
```
コンテナ内からホストのプロセスは確認できない
Namespaceによってプロセスの隔離がされている

```
♥ ❯ uname -a
Darwin ShotaronoMacBook-Air.local 22.3.0 Darwin Kernel Version 22.3.0: Mon Jan 30 20:38:43 PST 2023; root:xnu-8792.81.3~2/RELEASE_ARM64_T8112 arm64

♥ ❯ sudo docker run --rm -it fedora:38 bash
Password:
Unable to find image 'fedora:38' locally
2024/12/30 08:33:11 must use ASL logging (which requires CGO) if running as root
38: Pulling from library/fedora
ae9448aa32b5: Pull complete
Digest: sha256:b9ff6f23cceb5bde20bb1f79b492b98d71ef7a7ae518ca1b15b26661a11e6a94
Status: Downloaded newer image for fedora:38
[root@645cf9419e5d /]# uname -a
Linux 645cf9419e5d 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022 aarch64 GNU/Linux
```
ホストとコンテナでOSが異なる場合、ホストのOS情報が表示される
MacOSのホストでFedoraのコンテナを起動した場合、Linuxの情報が表示される(コンテナの中はlinuxkitが動いている？)

いつも`--rm`を使わないので、コンテナが残っていることがあることに気づいた

高レイヤランタイムと低レイヤランタイムの違いが書かれている

## 2章
コンテナの仕組みと要素技術

```
♥ ❯ curl --unix-socket /var/run/docker.sock http://v1.40/containers/json
[]
```
コンテナイメージがないので空の配列が返ってくる

```
♥ ❯ curl --unix-socket /var/run/docker.sock \
∙ -X POST \
∙ -H 'Content-Type: application/json' \
∙ 'http://v1.40/images/create?fromImage=ubuntu&tag=latest'
{"status":"Pulling from library/ubuntu","id":"latest"}
{"status":"Digest: sha256:80dd3c3b9c6cecb9f1667e9290b3bc61b78c2678c02cbdae5f0fea92cc6734ab"}
{"status":"Status: Image is up to date for ubuntu:latest"}
```
ubuntu:latestのイメージをunix socket経由でpullしている

```
♥ ❯ cat request.json
{
  "AttachStdin": false,
  "AttachStdout": true,
  "AttachStdeer": true,
  "Tty": true,
  "OpenStdin": false,
  "StdinOnce": false,
  "EntryPoint": "/bin/bash",
  "Image": "ubuntu:latest"
}

 ❯ curl --unix-socket /var/run/docker.sock \
-X POST \
-H 'Content-Type: application/json' \
∙ --data @request.json \
∙ http://v1.40/containers/create
{"Id":"d15a59e3d079547419273bb60226291bcf5c92aa3bad1931273147ce4158ecd0","Warnings":[]}
```
request.jsonの内容でコンテナを作成している

```
♥ ❯ docker ps -a
CONTAINER ID   IMAGE                                                                                  COMMAND                  CREATED          STATUS                       PORTS     NAMES
d15a59e3d079   ubuntu:latest                                                                          "/bin/bash"              57 seconds ago   Created                                jolly_jackson
```
Createdの状態でコンテナが作成されている

```
♥ ❯ curl --unix-socket /var/run/docker.sock \
∙ -X POST \
∙ -H 'Content-Type: application/json' \
∙ http:/v1.40/containers/d15a59e3d079/start
```
コンテナを起動している

```
♥ ❯ docker ps
CONTAINER ID   IMAGE           COMMAND       CREATED          STATUS         PORTS     NAMES
d15a59e3d079   ubuntu:latest   "/bin/bash"   12 minutes ago   Up 7 seconds             jolly_jackson

♥ ❯ curl --unix-socket /var/run/docker.sock http://v1.40/containers/json | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   891    0   891    0     0  50219      0 --:--:-- --:--:-- --:--:-- 52411
[
  {
    "Id": "d15a59e3d079547419273bb60226291bcf5c92aa3bad1931273147ce4158ecd0",
    "Names": [
      "/jolly_jackson"
    ],
    "Image": "ubuntu:latest",
    "ImageID": "sha256:20377134ad8875ad73c4a4f12b5d8e28c8665da80756c2afbb47d1f730bf2e5e",
    "Command": "/bin/bash",
    "Created": 1735517123,
    "Ports": [],
    "Labels": {
      "org.opencontainers.image.ref.name": "ubuntu",
      "org.opencontainers.image.version": "24.04"
    },
    "State": "running",
    "Status": "Up About a minute",
    "HostConfig": {
      "NetworkMode": "default"
    },
    "NetworkSettings": {
      "Networks": {
        "bridge": {
          "IPAMConfig": null,
          "Links": null,
          "Aliases": null,
          "NetworkID": "10f5a39f660340ee4d29c9aa6a9aed452485f8b0db983ce738185645d58ac6b7",
          "EndpointID": "b372952d69518667867a70c92b21c11b966a324791b73d524117d6b4eb22766e",
          "Gateway": "172.17.0.1",
          "IPAddress": "172.17.0.2",
          "IPPrefixLen": 16,
          "IPv6Gateway": "",
          "GlobalIPv6Address": "",
          "GlobalIPv6PrefixLen": 0,
          "MacAddress": "02:42:ac:11:00:02",
          "DriverOpts": null
        }
      }
    },
    "Mounts": []
  }
]
```

コンテナが起動していることが確認できる
エントリポイントの/bin/bashで起動している

```
❮  curl --unix-socket /var/run/docker.sock \
-X POST \
-H 'Content-Type: application/json' \
--data-binary '{"AttachStdin": true, "AttachStdout": true, "AttachStderr": true, "Cmd": ["uname", "-a"], "DetachKeys": "ctrl-p,ctrl-q", "Tty": true}' \
http://v1.40/containers/d15a59e3d079/exec
{"Id":"07f15f2d29c9fd5194c5634fcf558334940e9afd0389c5e8640cc56bd0a2e383"}

♥ ❯ curl -s --unix-socket /var/run/docker.sock \
∙ -X POST \
∙ -H 'Content-Type: application/json' \
∙ --data-binary '{"Detach": false, "Tty": false}' \
∙ http://v1.40/exec/07f15f2d29c9fd5194c5634fcf558334940e9afd0389c5e8640cc56bd0a2e383/start --output /tmp/output.txt

♥ ❯ cat /tmp/output.txt
sLinux d15a59e3d079 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022 aarch64 aarch64 aarch64 GNU/Linux
```

コンテナ内でuname -aを実行している
/container/d15a59e3d079/execでコンテナ内でコマンドを実行するためのexecオブジェクトを作成

```
♥ ❯ ls
 Dockerfile   file.txt

♥ ❯ cat Dockerfile
FROM alpine:latest

RUN apk update
RUN apk add curl

COPY file.txt /etc/file.txt

♥ ❯ docker build -t myimage:test .
```

Dockerfileを作成してイメージをビルドしている

```
♥ ❯ mkdir dump
♥ ❯ docker save myimage:test -o dump/myimage.tar
♥ ❯ ls dump
 myimage.tar
```

イメージをtarファイルに保存している

```
♥ ❯ cd dump
♥ ❯ tar xf myimage.tar
♥ ❯ rm myimage.tar
♥ ❯ tree .
.
├── 35ecf720e7387c604a1e5b9ccc7dab638775baf4578f02c362a97c2f6a40570b
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── 3cda78a28fb756c62a4d3e10eb3fced8a84fcf5a95c4f769c4b1279a1fa8b3c6.json
├── 55769501a47b4b57aa19ac868a940690a55b33aa457d88b72f932bf3af4e3103
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── 79c95b7729f975636f63648ba1e24fb3603bfea86f30fc749bbe30d2a3152384
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── 88dde31d66f812b0eed9bc1454fd8d7402b8e2810ed09faab0f42ea9ea3d0eea
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── manifest.json
└── repositories

5 directories, 15 files
```

tarファイルを展開している

```
♥ ❯ cat manifest.json | jq .
[
 {
   "Config": "3cda78a28fb756c62a4d3e10eb3fced8a84fcf5a95c4f769c4b1279a1fa8b3c6.json",
   "RepoTags": [
     "myimage:test"
   ],
   "Layers": [
     "79c95b7729f975636f63648ba1e24fb3603bfea86f30fc749bbe30d2a3152384/layer.tar",
     "35ecf720e7387c604a1e5b9ccc7dab638775baf4578f02c362a97c2f6a40570b/layer.tar",
     "88dde31d66f812b0eed9bc1454fd8d7402b8e2810ed09faab0f42ea9ea3d0eea/layer.tar",
     "55769501a47b4b57aa19ac868a940690a55b33aa457d88b72f932bf3af4e3103/layer.tar"
   ]
 }
]
```

manifest.jsonの内容を確認している

```
♥ ❯ cat 3cda78a28fb756c62a4d3e10eb3fced8a84fcf5a95c4f769c4b1279a1fa8b3c6.json
{"architecture":"arm64","config":{"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/sh"],"WorkingDir":"/","OnBuild":null},"created":"2024-12-30T00:37:45.330197751Z","history":[{"created":"2024-12-05T12:49:04Z","created_by":"ADD alpine-minirootfs-3.21.0-aarch64.tar.gz / # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2024-12-05T12:49:04Z","created_by":"CMD [\"/bin/sh\"]","comment":"buildkit.dockerfile.v0","empty_layer":true},{"created":"2024-12-30T00:37:44.349027959Z","created_by":"RUN /bin/sh -c apk update # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2024-12-30T00:37:45.316036251Z","created_by":"RUN /bin/sh -c apk add curl # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2024-12-30T00:37:45.330197751Z","created_by":"COPY file.txt /etc/file.txt # buildkit","comment":"buildkit.dockerfile.v0"}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:977340364f395522c48194db0cf2a81b7643d1bd1378a4c16dc848095a39de7d","sha256:72ef60dd848b3f5a7b1574d23cf60625e0597c82d5a6e4b9087d455d7952421c","sha256:87b2c0775611ce182efe35190374edb1a26ced8ac8e5b83f7e6e5cfab05d8539","sha256:a874f1a59e5cbe3e992e52399390f9a97a08b37f6e20c88f603bde89e79c5dc2"]},"variant":"v8"}%
```

Configで指定されたファイルの内容を確認している

ケーパビリティとは、プロセスに付与される権限のこと
Namespaceとは、プロセス間のリソースを隔離する仕組み
cgroupとは、リソースの制限を行う仕組み
Seccompとは、システムコールの制限を行う仕組み
LSMとは、セキュリティモジュールを管理する仕組み

## 3章
コンテナへの主要な攻撃ルート

- Docker APIへの攻撃
- エンドポイント端末で動作するDocker APIへの攻撃
- コンテナランタイムの脆弱性
- ケーパビリティの設定不備によるエスケープ
- CAP_NET_RAWによるコンテナネットワークの盗聴
- CAP_SYS_ADMINによる権限昇格
- CAP_DAC_READ_SEARCHによる権限昇格
- 特権コンテナの危険性と攻撃例
- DoS攻撃
- センシティブなファイルのマウント
- Linuxカーネルへの攻撃
- コンテナイメージやソフトウェアの脆弱性を利用した攻撃

## 4章
堅牢なコンテナイメージを作る

コンテナイメージのための脆弱性スキャナ
- Trivy

```
♥ ❯ trivy image python:3.4-alpine
```

今度から脆弱性のあったコンテナイメージを確認するときは`trivy`を使おう

```
♥ ❯ trivy image ubuntu:20.04
```

Ubuntuの20.04のイメージをスキャンしている

```
♥ ❯ trivy image --ignore-unfixed ubuntu:20.04
2024-12-30T13:32:56+09:00       INFO    [vuln] Vulnerability scanning is enabled
2024-12-30T13:32:56+09:00       INFO    [secret] Secret scanning is enabled
2024-12-30T13:32:56+09:00       INFO    [secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2024-12-30T13:32:56+09:00       INFO    [secret] Please see also https://aquasecurity.github.io/trivy/v0.58/docs/scanner/secret#recommendation for faster secret detection
2024-12-30T13:32:56+09:00       INFO    Detected OS     family="ubuntu" version="20.04"
2024-12-30T13:32:56+09:00       INFO    [ubuntu] Detecting vulnerabilities...   os_version="20.04" pkg_num=92
2024-12-30T13:32:56+09:00       INFO    Number of language-specific files       num=0

ubuntu:20.04 (ubuntu 20.04)

Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

`--ignore-unfixed`をつけると未修正の脆弱性を無視してスキャンする

```
♥ ❯ cat Dockerfile
FROM ubuntu:20.04

RUN apt-get update && apt-get install curl
EXPOSE 22

♥ ❯ trivy config .
```

Dockerfileの内容を確認している

OPAとは、ポリシーを管理するためのツール
https://www.openpolicyagent.org/

Syftとは、コンテナイメージの構成を確認するためのツール(SBOM)

SyftとGrypeを使ったSBOM生成もある

```
♥ ❯ cat Dockerfile
FROM alpine

RUN echo "THIS IS SECRET" > /secret.txt
RUN rm /secret.txt

♥ ❯ docker build -t test:latest .
♥ ❯ mkdir dump
♥ ❯ docker save test:latest -o dump/test.tar
♥ ❯ tar -xf dump/8532219fb8ab2de1ffc06c6ae4352e158cca1d273432dca623055b4c6bcb2641/layer.tar
♥ ❯ cat secret.txt
THIS IS SECRET
```

Dockerfileでシークレット情報を含むファイルを作成している

```
♥ ❯ cat Dockerfile
FROM alpine

RUN --mount=type=secret,id=mysecret,target=/secret.txt

♥ ❯ DOCKER_BUILDKIT=1 docker build =t test:latest --secret id=mysecret,src=$(pwd)/secret.txt .
```

シークレット情報を含むファイルをマウントしている

コンテナイメージへの署名による改ざん対策

Docker Hubのイメージを使用する場合は`verified_publisher`を使う

依存パッケージやライブラリが少ない`Distoless`を使う

## 5章
コンテナランタイムをセキュアに運用する

- ケーパビリティの制限
- システムコールの制限
- ファイルアクセスの制限
- AppArmorによるファイルアクセス制限
- リソースの制限
- メモリ使用量の制限
- プロセス数の制限
- ストレージ使用量の制限
- cpulimitとulimitを使ったリソース制限
- コンテナ実行ユーザーの変更と権限昇格の防止
- セキュアなコンテナランタイムの使用

## 6章
セキュアなコンテナ環境の構築

- コンテナのセキュリティ監視
  - ソフトウェアで監視
    - Sysdig
    - Falco
    - Fluentd/Fluent Bit
- コンテナのログ収集の設計
- コンテナの操作ログの記録
- セキュアに運用するためのガイドラインがある
  - CISベンチマーク
  - NIST SP800-190
  - OWASP Docker Security Cheat Sheet
- アプリケーションログやイメージレジストリの監視
