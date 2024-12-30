# lima tutorial

**Date:** 2024-12-31 04:12:43

Limaのドキュメントを読んでみて気になったことや何となくわかったことをメモ
[https://lima-vm.io/docs/]

## install
- brewでインストールできる
```
brew install lima
``` 

## 試してみる
- limactlが動かない

```
♥ ❯ limactl start
INFO[0000] Using the existing instance "default"
INFO[0000] Starting the instance "default" with VM driver "qemu"
FATA[0000] failed to find the QEMU binary for the architecture "aarch64": exec: "qemu-system-aarch64": executable file not found in $PATH
```
- qemuをインストールしたら動いた

```
❯ brew install qemu
```

- `limactl shell <INSTANCE> <COMMAND>`でインスタンスに対してLinuxコマンドを打てる
- デフォルトインスタンスなら`lima <COMMAND>`だけで良いらしい

## 使ってみる
- neofetch

```
♥ ❯ lima sudo apt-get install -y neofetch
♥ ❯ lima neofetch
            .-/+oossssoo+/-.               stanaka@lima-default
        `:+ssssssssssssssssss+:`           --------------------
      -+ssssssssssssssssssyyssss+-         OS: Ubuntu 24.10 aarch64
    .ossssssssssssssssssdMMMNysssso.       Host: QEMU Virtual Machine virt-9.2
   /ssssssssssshdmmNNmmyNMMMMhssssss/      Kernel: 6.11.0-12-generic
  +ssssssssshmydMMMMMMMNddddyssssssss+     Uptime: 6 mins
 /sssssssshNMMMyhhyyyyhmNMMMNhssssssss/    Packages: 771 (dpkg)
.ssssssssdMMMNhsssssssssshNMMMdssssssss.   Shell: bash 5.2.32
+sssshhhyNMMNyssssssssssssyNMMMysssssss+   Resolution: 1280x800
ossyNMMMNyMMhsssssssssssssshmmmhssssssso   Terminal: /dev/pts/0
ossyNMMMNyMMhsssssssssssssshmmmhssssssso   CPU: (4)
+sssshhhyNMMNyssssssssssssyNMMMysssssss+   GPU: 00:04.0 Red Hat, Inc. Virtio 1.0 GPU
.ssssssssdMMMNhsssssssssshNMMMdssssssss.   Memory: 231MiB / 3900MiB
 /sssssssshNMMMyhhyyyyhdNMMMNhssssssss/
  +sssssssssdmydMMMMMMMMddddyssssssss+
   /ssssssssssshdmNNNNmyNMMMMhssssss/
    .ossssssssssssssssssdMMMNysssso.
      -+sssssssssssssssssyyyssss+-
        `:+ssssssssssssssssss+:`
            .-/+oossssoo+/-.
```

- コンテナを立ち上げてみる

containerdはまた別の機会で学習する

```
♥ ❯ nerdctl.lima run -d --name nginx -p 127.0.0.1:8080:80 nginx:alpine
♥ ❯ curl http://127.0.0.1:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html
```

- GitHub Actionsを使用してUbuntu以外のHostにも対応できる
[https://lima-vm.io/docs/examples/gha/]

## Templates
- Tier表がついていて便利。Tier1は無難かな
[https://lima-vm.io/docs/templates/]

## よく使いそうなコマンド

- limactl create
インスタンスを作成する yamlファイルからもインスタンスを作成できる
[https://lima-vm.io/docs/reference/limactl_create/]

- limactl start
インスタンスを動かす
[https://lima-vm.io/docs/reference/limactl_start/]

- limactl edit
インスタンスかyamlファイルを修正する
[https://lima-vm.io/docs/reference/limactl_edit/]

- limactl shell
インスタンス内でシェルを実行する
[https://lima-vm.io/docs/reference/limactl_shell/]

- ssh
インスタンス内に入るには以下のコマンドを使う
```
ssh -F ~/.lima/default/ssh.config lima-default
```

`limactl ssh`はdeprecatedなので使わない


