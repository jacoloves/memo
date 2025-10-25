# kubernetes hardway memo

**Date:** 2025-10-17 05:39:05

- hostname変更
sudo hostnamectl set-hostname controller-1
- 設定適用
sudo netplan apply
- machine-idを再生成
sudo systemd-machine-id-setup
- SSHホストキーを再生成
sudo dpkg-reconfigure openssh-server

- for文でipぶん回すのインフラエンジニアっぽいw
- 全vm疎通OK
```
ubuntu@gateway-01:~$ for i in 10 11 12 20 21 22; do echo "Testing 192.168.8.$i"; ping -c 2 192.168.8.$i; done
Testing 192.168.8.10
PING 192.168.8.10 (192.168.8.10) 56(84) bytes of data.
64 bytes from 192.168.8.10: icmp_seq=1 ttl=64 time=0.190 ms
64 bytes from 192.168.8.10: icmp_seq=2 ttl=64 time=0.299 ms

--- 192.168.8.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1030ms
rtt min/avg/max/mdev = 0.190/0.244/0.299/0.054 ms
Testing 192.168.8.11
PING 192.168.8.11 (192.168.8.11) 56(84) bytes of data.
64 bytes from 192.168.8.11: icmp_seq=1 ttl=64 time=0.141 ms
64 bytes from 192.168.8.11: icmp_seq=2 ttl=64 time=0.222 ms

--- 192.168.8.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.141/0.181/0.222/0.040 ms
Testing 192.168.8.12
PING 192.168.8.12 (192.168.8.12) 56(84) bytes of data.
64 bytes from 192.168.8.12: icmp_seq=1 ttl=64 time=0.158 ms
64 bytes from 192.168.8.12: icmp_seq=2 ttl=64 time=0.272 ms

--- 192.168.8.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1021ms
rtt min/avg/max/mdev = 0.158/0.215/0.272/0.057 ms
Testing 192.168.8.20
PING 192.168.8.20 (192.168.8.20) 56(84) bytes of data.
64 bytes from 192.168.8.20: icmp_seq=1 ttl=64 time=0.157 ms
64 bytes from 192.168.8.20: icmp_seq=2 ttl=64 time=0.249 ms

--- 192.168.8.20 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1021ms
rtt min/avg/max/mdev = 0.157/0.203/0.249/0.046 ms
Testing 192.168.8.21
PING 192.168.8.21 (192.168.8.21) 56(84) bytes of data.
64 bytes from 192.168.8.21: icmp_seq=1 ttl=64 time=0.153 ms
64 bytes from 192.168.8.21: icmp_seq=2 ttl=64 time=0.335 ms

--- 192.168.8.21 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1021ms
rtt min/avg/max/mdev = 0.153/0.244/0.335/0.091 ms
Testing 192.168.8.22
PING 192.168.8.22 (192.168.8.22) 56(84) bytes of data.
64 bytes from 192.168.8.22: icmp_seq=1 ttl=64 time=0.164 ms
64 bytes from 192.168.8.22: icmp_seq=2 ttl=64 time=0.242 ms

--- 192.168.8.22 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1021ms
rtt min/avg/max/mdev = 0.164/0.203/0.242/0.039 ms
```

[Q] gateway-01のみローカルPCからcontrollerやworkerノードに接続できるのは二つのネットワークデバイスを持っているから？

- 全てのペインに同じコマンドを送信できて便利
```
[prefix]Ctrl+b then :
setw synchronize-panes on
```

- cfsslとは
[cfssl](https://github.com/cloudflare/cfssl)
秘密鍵ファイルや証明書ファイルの発行に必要なファイルを作成してくれる
[https://zenn.dev/hiro345/articles/20241201_advent_calendar_2024_14]

## 3章 Compute Resourcesの確認

### 概要
1. ネットワーク設定
2. 各VMのIPアドレス
3. ファイアウォール設定
4. VM間の通信

- VM間のIP
```
for node in controller-0 controller-1 controller-2 worker-0 worker-1 worker-2; do
  case $node in
    controller-0) ip="192.168.8.10" ;;
    controller-1) ip="192.168.8.11" ;;
    controller-2) ip="192.168.8.12" ;;
    worker-0) ip="192.168.8.20" ;;
    worker-1) ip="192.168.8.21" ;;
    worker-2) ip="192.168.8.22" ;;
  esac

  echo "=== Checking $node ($ip) ==="
  ssh ubuntu@$ip "hostname && ip addr show ens18 | grep 'inet '"
  echo ""
done

---

=== Checking controller-0 (192.168.8.10) ===
controller-0
    inet 192.168.8.10/24 brd 192.168.8.255 scope global ens18

=== Checking controller-1 (192.168.8.11) ===
controller-1
    inet 192.168.8.11/24 brd 192.168.8.255 scope global ens18

=== Checking controller-2 (192.168.8.12) ===
controller-2
    inet 192.168.8.12/24 brd 192.168.8.255 scope global ens18

=== Checking worker-0 (192.168.8.20) ===
worker-0
    inet 192.168.8.20/24 brd 192.168.8.255 scope global ens18

=== Checking worker-1 (192.168.8.21) ===
worker-1
    inet 192.168.8.21/24 brd 192.168.8.255 scope global ens18

=== Checking worker-2 (192.168.8.22) ===
worker-2
    inet 192.168.8.22/24 brd 192.168.8.255 scope global ens18

```

- 各ノード間の通信テスト
```
ssh ubuntu@192.168.8.10 "
echo 'Testing from controller-0:'
for ip in 192.168.8.1 192.168.8.11 192.168.8.12 192.168.8.20 192.168.8.21 192.168.8.22; do
  ping -c 1 -W 1 \$ip > /dev/null 2>&1 && echo \"✓ \$ip reachable\" || echo \"x \$ip unreachable\"
done
"

---

Testing from controller-0:
✓ 192.168.8.1 reachable
✓ 192.168.8.11 reachable
✓ 192.168.8.12 reachable
✓ 192.168.8.20 reachable
✓ 192.168.8.21 reachable
✓ 192.168.8.22 reachable

```

- NAT動作確認

```
for ip in 192.168.8.10 192.168.8.11 192.168.8.12 192.168.8.20 192.168.8.21 192.168.8.22; do
  echo "Testing internet from $ip:"
  ssh ubuntu@$ip "ping -c 2 8.8.8.8 > /dev/null 2>&1 && echo '✓ Internet OK' || 'x Internet NG'"
done

---

Testing internet from 192.168.8.10:
✓ Internet OK
Testing internet from 192.168.8.11:
✓ Internet OK
Testing internet from 192.168.8.12:
✓ Internet OK
Testing internet from 192.168.8.20:
✓ Internet OK
Testing internet from 192.168.8.21:
✓ Internet OK
Testing internet from 192.168.8.22:
✓ Internet OK


```

- 各VMの詳細確認
```
for ip in 192.168.8.10 192.168.8.11 192.168.8.12 192.168.8.20 192.168.8.21 192.168.8.22; do
  echo "=== VM: $ip ==="
  ssh ubuntu@$ip "
    echo 'Hostname: \$(hostname)'
    echo 'IP: \$(ip addr show ens18 | grep \"inet \" | awk \"{print \\\$2}\")'
    echo 'CPU Cores: \$(nproc)'
    echo 'Memory: \$(free -h | grep Mem | awk \"{print \\\$2}\")'
    echo 'Swap: \$(free -h | grep Swap | awk \"{print \\\$2}\")'
    echo 'Disk: \$(df -h / | tail -1 | awk \"{print \\\$2}\")'
  "
  echo ""
done

```


- gateway-01のFWルール
```
echo "=== NAT Rules ==="
sudo iptables -t nat -L POSTROUTING -n -v | grep MASQUERADE

echo -e "\n=== FORWARD Rules ==="
sudo iptables -L FORWARD -n -v | grep 192.168.8

echo -e "\n=== INPUT Rules (Allowed Ports) ==="
sudo iptables -L INPUT -n -v | grep -E "dpt:(22|80|443|6443)"


---

=== NAT Rules ===
[sudo] password for ubuntu:
  624 45367 MASQUERADE  all  --  *      ens18   192.168.8.0/24       0.0.0.0/0

=== FORWARD Rules ===
 162K 9482K ACCEPT     all  --  *      *       192.168.8.0/24       0.0.0.0/0
 185K  797M ACCEPT     all  --  *      *       0.0.0.0/0            192.168.8.0/24

=== INPUT Rules (Allowed Ports) ===
 3427  259K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:6443

```

- /etc/hostsファイルの確認
```
ssh ubuntu@192.168.8.10 "cat /etc/hosts | grep -E '192.168.8'"

---

192.168.8.1  gateway-01
192.168.8.10 controller-0
192.168.8.11 controller-1
192.168.8.12 controller-2
192.168.8.20 worker-0
192.168.8.21 worker-1
192.168.8.22 worker-2
```

- 全ノードで一括確認
```
for ip in 192.168.8.10 192.168.8.11 192.168.8.12 192.168.8.20 192.168.8.21 192.168.8.22; do
  echo "=== /etc/hosts on $ip ==="
  ssh ubuntu@$ip "grep -E '192.168.8' /etc/hosts"
  echo ""
done

---

=== /etc/hosts on 192.168.8.10 ===
192.168.8.1  gateway-01
192.168.8.10 controller-0
192.168.8.11 controller-1
192.168.8.12 controller-2
192.168.8.20 worker-0
192.168.8.21 worker-1
192.168.8.22 worker-2

=== /etc/hosts on 192.168.8.11 ===
192.168.8.1  gateway-01
192.168.8.10 controller-0
192.168.8.11 controller-1
192.168.8.12 controller-2
192.168.8.20 worker-0
192.168.8.21 worker-1
192.168.8.22 worker-2

=== /etc/hosts on 192.168.8.12 ===
192.168.8.1  gateway-01
192.168.8.10 controller-0
192.168.8.11 controller-1
192.168.8.12 controller-2
192.168.8.20 worker-0
192.168.8.21 worker-1
192.168.8.22 worker-2

=== /etc/hosts on 192.168.8.20 ===
192.168.8.1  gateway-01
192.168.8.10 controller-0
192.168.8.11 controller-1
192.168.8.12 controller-2
192.168.8.20 worker-0
192.168.8.21 worker-1
192.168.8.22 worker-2

=== /etc/hosts on 192.168.8.21 ===
192.168.8.1  gateway-01
192.168.8.10 controller-0
192.168.8.11 controller-1
192.168.8.12 controller-2
192.168.8.20 worker-0
192.168.8.21 worker-1
192.168.8.22 worker-2

=== /etc/hosts on 192.168.8.22 ===
192.168.8.1  gateway-01
192.168.8.10 controller-0
192.168.8.11 controller-1
192.168.8.12 controller-2
192.168.8.20 worker-0
192.168.8.21 worker-1
192.168.8.22 worker-2

```

- 全体確認バッチ
```
#!/bin/bash

echo "=========================================="
echo "Kubernetes Cluster Health Check"
echo "=========================================="

for ip in 192.168.8.10 192.168.8.11 192.168.8.12 192.168.8.20 192.168.8.21 192.168.8.22; do
  echo -e "\n--- Node: $ip ---"

  hostname=$(ssh ubuntu@$ip "hostname" 2>/dev/null)
  echo "Hostname: $hostname"

  node_ip=$(ssh ubuntu@ip "ip addr show ens18 | grep 'inet' ' | awsk '{print \$2}'" 2>/dev/null)
  echo "IP: $node_ip"

  internet=$(ssh ubuntu@ip "ping -c 1 -W 1 8.8.8.8 > /dev/null 2>&1 && echo 'OK' || echo 'NG'" 2>/dev/null)
  echo "Internet: $internet"

  swap=$(ssh ubuntu@ip "free -h | grep Swap | awk '{print \$2}'" 2>/dev/null)
  echo "Swap: $swap"
done


echo -e "\n=========================================="
echo "Health Check Complete!"
echo "=========================================="
```

## 4章 Certificate AuthorityとTLS証明書の生成

### 概要
このステップでは以下を実施します：
1. Certificate Authority (CA) の作成
2. コンポーネント用のTLS証明書の生成：
  - Admin用クライアント証明書
  - Kubelet用クライアント証明書（各ワーカーノード）
  - Controller Manager用クライアント証明書
  - Kube Proxy用クライアント証明書
  - Scheduler用クライアント証明書
  - Kubernetes APIサーバー用証明書
  - Service Account用キーペア

### なぜ証明書が必要？
Kubernetesのコンポーネント間通信は全て暗号化され、相互に認証される必要があります：
- APIサーバー ↔ etcd
- APIサーバー ↔ kubelet
- kubectl ↔ APIサーバー
- など

- CA設定ファイルの作成
```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

- CA証明書署名リクエスト（CSR）の作成
```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "Key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Tokyo",
    }
  ]
}
EOF

```

- CA証明書と秘密鍵を生成
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

- Admin用クライアント証明書
```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

```

- 証明書を作成
```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

```

- Kubelet用クライアント証明書
- worker-0
```
cat > worker-0-csr.json <<EOF
{
  "CN": "system:node:worker-0",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=worker-0,192.168.8.20 \
  -profile=kubernetes \
  worker-0-csr.json | cfssljson -bare worker-0

```

- worker-1
```
cat > worker-1-csr.json <<EOF
{
  "CN": "system:node:worker-1",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=worker-1,192.168.8.21 \
  -profile=kubernetes \
  worker-1-csr.json | cfssljson -bare worker-1

```

- worker-2
```
cat > worker-2-csr.json <<EOF
{
  "CN": "system:node:worker-2",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=worker-2,192.168.8.22 \
  -profile=kubernetes \
  worker-2-csr.json | cfssljson -bare worker-2

```

- Controller Manager用クライアント証明書
```
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

```

- Kube Proxy用クライアント証明書
```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

```

- Scheduler用クライアント証明書
```
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

```

- Kubernetes APIサーバー用証明書
```
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,192.168.8.10,192.168.8.11,192.168.8.12,192.168.8.1,192.168.3.18,127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

```

- Service Account用キーペア
```
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

```

## 5章 Kubernetes Configuration Files

### 概要
kubeconfigファイルは、Kubernetesクラスターへの接続情報を含む設定ファイルです。
各コンポーネントが使用するkubeconfigには以下の情報が含まれます：
- クラスター情報（APIサーバーのアドレス、CA証明書）
- ユーザー認証情報（クライアント証明書、秘密鍵）
- コンテキスト（どのクラスターにどのユーザーで接続するか）


- Kubernetes APIサーバーのアドレス設定
```
KUBERNETES_PUBLIC_ADDRESS=192.168.3.18
```

- kubelet用kubeconfigの生成
- worker-0用
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=worker-0.kubeconfig

kubectl config set-credentials system:node:worker-0 \
  --client-certificate=worker-0.pem \
  --client-key=worker-0-key.pem \
  --embed-certs=true \
  --kubeconfig=worker-0.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:worker-0 \
  --kubeconfig=worker-0.kubeconfig

kubectl config use-context default --kubeconfig=worker-0.kubeconfig
```

- worker-1用
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=worker-1.kubeconfig

kubectl config set-credentials system:node:worker-1 \
  --client-certificate=worker-1.pem \
  --client-key=worker-1-key.pem \
  --embed-certs=true \
  --kubeconfig=worker-1.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:worker-1 \
  --kubeconfig=worker-1.kubeconfig

kubectl config use-context default --kubeconfig=worker-1.kubeconfig
```

- worker-2用
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=worker-2.kubeconfig

kubectl config set-credentials system:node:worker-2 \
  --client-certificate=worker-2.pem \
  --client-key=worker-2-key.pem \
  --embed-certs=true \
  --kubeconfig=worker-2.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:worker-2 \
  --kubeconfig=worker-2.kubeconfig

kubectl config use-context default --kubeconfig=worker-2.kubeconfig
```

- kube-proxy用kubeconfigの生成
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

- kube-controller-manager用kubeconfigの生成
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

- kube-scheduler用kubeconfigの生成
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

- admin用kubeconfigの生成
```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

## 6章 Data Encryption Config and Key

### 概要
Kubernetesは、etcdに保存される機密情報（Secrets）を暗号化できます。これにより：
- etcdのバックアップが漏洩しても、Secretの内容は保護される
- etcdに直接アクセスされても、データは暗号化されている

- 暗号化キーの生成
```
ENCPYPTION_KEY=$(head -c 32 /dev/urandom | base64)

echo $ENCPYPTION_KEY
```

- 暗号化設定ファイルの作成
```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCPYPTION_KEY}
      - identity: {}
EOF

```

- 完了確認
```
echo "=== Generated Files Summary ==="
echo "Certificates: $(ls -l *.pem | wc -l) files"
echo "Kubeconfigs: $(ls -l *.kubeconfig | wc -l) files"
echo "Encryption Config: $(ls -l encryption-config.yaml | wc -l) files"
echo ""
echo "Total files ready for distribution!"

```

## 7章 Bootstrapping the etcd Cluster

### 概要
etcdは、Kubernetesの全てのクラスター状態を保存する分散キーバリューストアです。
このステップでは：
1. 必要なファイルをcontrollerノードに配布
2. etcdバイナリのダウンロード・インストール
3. 3ノード構成の高可用性etcdクラスターを構築

- 証明書とファイルの配布
```
# controller-0への配布
scp ca.pem kubernetes-key.pem kubernetes.pem ubuntu@192.168.8.10:~/


# controller-1への配布
scp ca.pem kubernetes-key.pem kubernetes.pem ubuntu@192.168.8.11:~/

# controller-2への配布
scp ca.pem kubernetes-key.pem kubernetes.pem ubuntu@192.168.8.12:~/

---

ubuntu@gateway-01:~/k8s-certs$ scp ca.pem kubernetes-key.pem kubernetes.pem ubuntu@192.168.8.10:~/
ca.pem                                                                           100% 1306     1.9MB/s   00:00
kubernetes-key.pem                                                               100% 1679     3.6MB/s   00:00
kubernetes.pem                                                                   100% 1659     4.4MB/s   00:00
ubuntu@gateway-01:~/k8s-certs$ scp ca.pem kubernetes-key.pem kubernetes.pem ubuntu@192.168.8.11:~/
ca.pem                                                                           100% 1306     3.4MB/s   00:00
kubernetes-key.pem                                                               100% 1679     5.6MB/s   00:00
kubernetes.pem                                                                   100% 1659     7.9MB/s   00:00
ubuntu@gateway-01:~/k8s-certs$ scp ca.pem kubernetes-key.pem kubernetes.pem ubuntu@192.168.8.12:~/
ca.pem                                                                           100% 1306     3.6MB/s   00:00
kubernetes-key.pem                                                               100% 1679     6.7MB/s   00:00
kubernetes.pem                                                                   100% 1659     7.0MB/s   00:00
ubuntu@gateway-01:~/k8s-certs$

```

- ウィンドウを分割して、etcdバイナリのダウンロードとインストール
```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.5.12/etcd-v3.5.12-linux-amd64.tar.gz"

tar -xvf etcd-v3.5.12-linux-amd64.tar.gz

sudo mv etcd-v3.5.12-linux-amd64/etcd* /usr/local/bin/

etcd --version
etcdctl version

---

# controller-0の出力

ubuntu@controller-0:~$ wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.5.12/etcd-v3.5.12-linux-amd64.tar.gz"
etcd-v3.5.12-linux-amd64.tar 100%[=============================================>]  19.40M  17.9MB/s    in 1.1s
ubuntu@controller-0:~$ ll
total 19912
drwxr-x--- 4 ubuntu ubuntu     4096 Oct 20 21:54 ./
drwxr-xr-x 3 root   root       4096 Oct 14 21:45 ../
-rw------- 1 ubuntu ubuntu      827 Oct 17 22:11 .bash_history
-rw-r--r-- 1 ubuntu ubuntu      220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 ubuntu ubuntu     3771 Jan  6  2022 .bashrc
drwx------ 2 ubuntu ubuntu     4096 Oct 14 21:45 .cache/
-rw-rw-r-- 1 ubuntu ubuntu     1306 Oct 20 21:46 ca.pem
-rw-rw-r-- 1 ubuntu ubuntu 20337842 Jan 31  2024 etcd-v3.5.12-linux-amd64.tar.gz
-rw------- 1 ubuntu ubuntu     1679 Oct 20 21:46 kubernetes-key.pem
-rw-rw-r-- 1 ubuntu ubuntu     1659 Oct 20 21:46 kubernetes.pem
-rw-r--r-- 1 ubuntu ubuntu      807 Jan  6  2022 .profile
drwx------ 2 ubuntu ubuntu     4096 Oct 14 21:45 .ssh/
-rw-r--r-- 1 ubuntu ubuntu        0 Oct 14 21:50 .sudo_as_admin_successful
-rw-rw-r-- 1 ubuntu ubuntu      165 Oct 20 21:54 .wget-hsts
ubuntu@controller-0:~$ tar -xvf etcd-v3.5.12-linux-amd64.tar.gz
etcd-v3.5.12-linux-amd64/
etcd-v3.5.12-linux-amd64/README.md
etcd-v3.5.12-linux-amd64/READMEv2-etcdctl.md
etcd-v3.5.12-linux-amd64/etcdutl
etcd-v3.5.12-linux-amd64/etcdctl
etcd-v3.5.12-linux-amd64/Documentation/
etcd-v3.5.12-linux-amd64/Documentation/README.md
etcd-v3.5.12-linux-amd64/Documentation/dev-guide/
etcd-v3.5.12-linux-amd64/Documentation/dev-guide/apispec/
etcd-v3.5.12-linux-amd64/Documentation/dev-guide/apispec/swagger/
etcd-v3.5.12-linux-amd64/Documentation/dev-guide/apispec/swagger/v3election.swagger.json
etcd-v3.5.12-linux-amd64/Documentation/dev-guide/apispec/swagger/rpc.swagger.json
etcd-v3.5.12-linux-amd64/Documentation/dev-guide/apispec/swagger/v3lock.swagger.json
etcd-v3.5.12-linux-amd64/README-etcdutl.md
etcd-v3.5.12-linux-amd64/README-etcdctl.md
etcd-v3.5.12-linux-amd64/etcd
ubuntu@controller-0:~$ sudo mv etcd-v3.5.12-linux-amd64/etcd* /usr/local/bin/
[sudo] password for ubuntu:
ubuntu@controller-0:~$ etcd --version
etcdctl version
etcd Version: 3.5.12
Git SHA: e7b3bb6cc
Go Version: go1.20.13
Go OS/Arch: linux/amd64
etcdctl version: 3.5.12
API version: 3.5
ubuntu@controller-0:~$

```

- etcdの設定
```
# 設定ディレクトリとデータディレクトリの作成
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd

# 証明書のコピー
sudo cp ~/ca.pem ~/kubernetes-key.pem ~/kubernetes.pem /etc/etcd/

---
# controller-0の出力　cpで途中入力を求められたが無事に/etc/etcdにコピーされている
ubuntu@controller-0:~$ sudo mkdir -p /etc/etcd /var/lib/etcd
ubuntu@controller-0:~$ sudo chmod 700 /var/lib/etcd
ubuntu@controller-0:~$ sudo cp ~/ca.pem ~/kubernetes-key.pem ~/kubernetes.pem /etc/etcd/
>
> ^C
ubuntu@controller-0:~$ ll
total 19916
drwxr-x--- 5 ubuntu ubuntu     4096 Oct 20 21:55 ./
drwxr-xr-x 3 root   root       4096 Oct 14 21:45 ../
-rw------- 1 ubuntu ubuntu      827 Oct 17 22:11 .bash_history
-rw-r--r-- 1 ubuntu ubuntu      220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 ubuntu ubuntu     3771 Jan  6  2022 .bashrc
drwx------ 2 ubuntu ubuntu     4096 Oct 14 21:45 .cache/
-rw-rw-r-- 1 ubuntu ubuntu     1306 Oct 20 21:46 ca.pem
drwxr-xr-x 3 ubuntu ubuntu     4096 Oct 20 21:55 etcd-v3.5.12-linux-amd64/
-rw-rw-r-- 1 ubuntu ubuntu 20337842 Jan 31  2024 etcd-v3.5.12-linux-amd64.tar.gz
-rw------- 1 ubuntu ubuntu     1679 Oct 20 21:46 kubernetes-key.pem
-rw-rw-r-- 1 ubuntu ubuntu     1659 Oct 20 21:46 kubernetes.pem
-rw-r--r-- 1 ubuntu ubuntu      807 Jan  6  2022 .profile
drwx------ 2 ubuntu ubuntu     4096 Oct 14 21:45 .ssh/
-rw-r--r-- 1 ubuntu ubuntu        0 Oct 14 21:50 .sudo_as_admin_successful
-rw-rw-r-- 1 ubuntu ubuntu      165 Oct 20 21:54 .wget-hsts
ubuntu@controller-0:~$ ll /etc/etcd
total 20
drwxr-xr-x  2 root root 4096 Oct 20 21:59 ./
drwxr-xr-x 97 root root 4096 Oct 20 21:59 ../
-rw-r--r--  1 root root 1306 Oct 20 21:59 ca.pem
-rw-------  1 root root 1679 Oct 20 21:59 kubernetes-key.pem
-rw-r--r--  1 root root 1659 Oct 20 21:59 kubernetes.pem
ubuntu@controller-0:~$

```

- etcdのsystemdサービスファイル作成
同期モードは一時解除

- controller-0での作業
```
INTERNAL_IP=192.168.8.10
ETCD_NAME=controller-0

cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.cm/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://192.168.8.10:2380,controller-1=https://192.168.8.11:2380,controller-2=https://192.168.8.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- controller-1での作業
```
INTERNAL_IP=192.168.8.11
ETCD_NAME=controller-1

cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.cm/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://192.168.8.10:2380,controller-1=https://192.168.8.11:2380,controller-2=https://192.168.8.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- controller-2での作業
```
INTERNAL_IP=192.168.8.12
ETCD_NAME=controller-2

cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.cm/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://192.168.8.10:2380,controller-1=https://192.168.8.11:2380,controller-2=https://192.168.8.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- etcdサービスの起動
同期モードを有効化しておく

- 全controllerノードで同時実行
```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd

---
#controller-0の出力

ubuntu@controller-0:~$ sudo systemctl daemon-reload
ubuntu@controller-0:~$ sudo systemctl enable etcd
Created symlink /etc/systemd/system/multi-user.target.wants/etcd.service → /etc/systemd/system/etcd.service.
ubuntu@controller-0:~$ sudo systemctl start etcd
ubuntu@controller-0:~$ sudo systemctl status etcd
● etcd.service - etcd
     Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2025-10-20 22:33:49 UTC; 26s ago
       Docs: https://github.cm/coreos
   Main PID: 2285 (etcd)
      Tasks: 8 (limit: 2218)
     Memory: 12.8M
        CPU: 433ms
     CGroup: /system.slice/etcd.service
             └─2285 /usr/local/bin/etcd --name controller-0 --cert-file=/etc/etcd/kubernetes.pem --key-file=/etc/e>

Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.03866Z","caller":"etcdserver/se>
Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.040276Z","caller":"embed/serve.>
Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.040356Z","caller":"etcdmain/mai>
Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.040454Z","caller":"etcdmain/mai>
Oct 20 22:33:49 controller-0 systemd[1]: Started etcd.
Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.046876Z","caller":"embed/serve.>
Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.054099Z","caller":"etcdserver/s>
Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.058485Z","caller":"membership/c>
Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.058665Z","caller":"api/capabili>
Oct 20 22:33:49 controller-0 etcd[2285]: {"level":"info","ts":"2025-10-20T22:33:49.058807Z","caller":"etcdserver/s>
ubuntu@controller-0:~$

```

- etcdクラスターの確認
```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem

---
# controller-0の出力
ubuntu@controller-0:~$ sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
286ea08a76969776, started, controller-1, https://192.168.8.11:2380, https://192.168.8.11:2379, false
671e6154d5107277, started, controller-2, https://192.168.8.12:2380, https://192.168.8.12:2379, false
7e87d57c936a3019, started, controller-0, https://192.168.8.10:2380, https://192.168.8.10:2379, false
ubuntu@controller-0:~$

```

## 8章 Kubernetes Control Planeの構築

### 概要
このステップでは、3つのcontrollerノードに以下のコンポーネントをインストールします：
1. kube-apiserver - Kubernetes APIのフロントエンド
2. kube-controller-manager - クラスターの状態を管理
3. kube-scheduler - Podの配置を決定

- 必要なファイルの配布
```
# controller-0への配布
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  ubuntu@192.168.8.10:~/


# controller-1への配布
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  ubuntu@192.168.8.11:~/


# controller-2への配布
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  ubuntu@192.168.8.12:~/

---

ubuntu@gateway-01:~/k8s-certs$ scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  ubuntu@192.168.8.10:~/

ca.pem                                                                           100% 1306     1.6MB/s   00:00
ca-key.pem                                                                       100% 1675     3.4MB/s   00:00
kubernetes-key.pem                                                               100% 1679     4.4MB/s   00:00
kubernetes.pem                                                                   100% 1659     4.7MB/s   00:00
service-account-key.pem                                                          100% 1675     4.1MB/s   00:00
service-account.pem                                                              100% 1428     4.0MB/s   00:00
encryption-config.yaml                                                           100%  240   582.7KB/s   00:00
kube-controller-manager.kubeconfig                                               100% 6355    14.9MB/s   00:00
kube-scheduler.kubeconfig                                                        100% 6305    13.9MB/s   00:00
ubuntu@gateway-01:~/k8s-certs$ scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  ubuntu@192.168.8.11:~/

ca.pem                                                                           100% 1306     3.5MB/s   00:00
ca-key.pem                                                                       100% 1675     6.4MB/s   00:00
kubernetes-key.pem                                                               100% 1679     6.2MB/s   00:00
kubernetes.pem                                                                   100% 1659     6.8MB/s   00:00
service-account-key.pem                                                          100% 1675     8.2MB/s   00:00
service-account.pem                                                              100% 1428     6.9MB/s   00:00
encryption-config.yaml                                                           100%  240     1.3MB/s   00:00
kube-controller-manager.kubeconfig                                               100% 6355    24.7MB/s   00:00
kube-scheduler.kubeconfig                                                        100% 6305    28.0MB/s   00:00
ubuntu@gateway-01:~/k8s-certs$ scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  ubuntu@192.168.8.12:~/

ca.pem                                                                           100% 1306     3.6MB/s   00:00
ca-key.pem                                                                       100% 1675     6.5MB/s   00:00
kubernetes-key.pem                                                               100% 1679     7.3MB/s   00:00
kubernetes.pem                                                                   100% 1659     7.9MB/s   00:00
service-account-key.pem                                                          100% 1675     8.2MB/s   00:00
service-account.pem                                                              100% 1428     7.2MB/s   00:00
encryption-config.yaml                                                           100%  240     1.3MB/s   00:00
kube-controller-manager.kubeconfig                                               100% 6355    25.2MB/s   00:00
kube-scheduler.kubeconfig                                                        100% 6305    28.1MB/s   00:00
ubuntu@gateway-01:~/k8s-certs$

```  a

- Kubernetesバイナリのダウンロードとインストール
同期モードを有効化しておく
- Kubernetesバイナリのダウンロード
```
sudo mkdir -p /etc/kubernetes/config

# Kubernetes v1.29.1バイナリ
wget -q --show-progress --https-only --timestamping \
  "https://dl.k8s.io/v1.29.1/bin/linux/amd64/kube-apiserver" \
  "https://dl.k8s.io/v1.29.1/bin/linux/amd64/kube-controller-manager" \
  "https://dl.k8s.io/v1.29.1/bin/linux/amd64/kube-scheduler" \
  "https://dl.k8s.io/v1.29.1/bin/linux/amd64/kubectl" 

---
# controller-0の出力

ubuntu@controller-0:~$ sudo mkdir -p /etc/kubernetes/config
[sudo] password for ubuntu:
ubuntu@controller-0:~$ ll /etc/kubernetes/config/
total 8
drwxr-xr-x 2 root root 4096 Oct 21 22:21 ./
drwxr-xr-x 3 root root 4096 Oct 21 22:21 ../
ubuntu@controller-0:~$ wget -q --show-progress --https-only --timestamping \
  "https://dl.k8s.io/v1.29.1/bin/linux/amd64/kube-apiserver" \
  "https://dl.k8s.io/v1.29.1/bin/linux/amd64/kube-controller-manager" \
  "https://dl.k8s.io/v1.29.1/bin/linux/amd64/kube-scheduler" \
  "https://dl.k8s.io/v1.29.1/bin/linux/amd64/kubectl"

kube-apiserver               100%[=============================================>] 117.99M  3.11MB/s    in 41s
kube-controller-manager      100%[=============================================>] 112.87M  2.21MB/s    in 42s
kube-scheduler               100%[=============================================>]  53.35M  3.06MB/s    in 19s
kubectl                      100%[=============================================>]  47.40M  5.32MB/s    in 8.2s
ubuntu@controller-0:~$


```

- バイナリのインストール
```
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl

sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/

kubectl version --client

---

# controller-0の出力
ubuntu@controller-0:~$ chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
ubuntu@controller-0:~$ sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
ubuntu@controller-0:~$ kubectl version --client
Client Version: v1.29.1
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
ubuntu@controller-0:~$

```

- kube-apiserverの設定
- 証明書と設定ファイルの配置
```
sudo mkdir -p /var/lib/kubernetes/

sudo cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/

---
# controller-0の出力

ubuntu@controller-0:~$ sudo mkdir -p /var/lib/kubernetes/
ubuntu@controller-0:~$ sudo cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/

ubuntu@controller-0:~$ ll /var/lib/kubernetes/
total 36
drwxr-xr-x  2 root root 4096 Oct 21 22:37 ./
drwxr-xr-x 42 root root 4096 Oct 21 22:37 ../
-rw-------  1 root root 1675 Oct 21 22:37 ca-key.pem
-rw-r--r--  1 root root 1306 Oct 21 22:37 ca.pem
-rw-------  1 root root  240 Oct 21 22:37 encryption-config.yaml
-rw-------  1 root root 1679 Oct 21 22:37 kubernetes-key.pem
-rw-r--r--  1 root root 1659 Oct 21 22:37 kubernetes.pem
-rw-------  1 root root 1675 Oct 21 22:37 service-account-key.pem
-rw-r--r--  1 root root 1428 Oct 21 22:37 service-account.pem
ubuntu@controller-0:~$

```

- kube-apiserver systemdサービスファイルの作成
- controller-0での作業
```
INTERNAL_IP=192.168.8.10

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.8.10:2379,https://192.168.8.11:2379,https://192.168.8.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${INTERNAL_IP}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- controller-1での作業
```
INTERNAL_IP=192.168.8.11

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.8.10:2379,https://192.168.8.11:2379,https://192.168.8.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${INTERNAL_IP}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- controller-2での作業
```
INTERNAL_IP=192.168.8.12

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.8.10:2379,https://192.168.8.11:2379,https://192.168.8.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${INTERNAL_IP}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- kube-controller-managerの設定
同期モードを有効化しておく

- kubeconfigの配置
```
sudo cp ~/kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

- kube-controller-manager systemdサービスファイルの作成
```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```

** 重要なパラメータ **
- `--cluster-cidr=10.200.0.0/16`: Pod用のIPレンジ
- `--service-cluster-ip-range=10.32.0.0/24`: Service用のIPレン
- `--leader-elect=true`: 高可用性のためのリーダー選出

- kube-schedulerの設定
- kubeconfigの配置
```
sudo cp ~/kube-scheduler.kubeconfig /var/lib/kubernetes/
```

- kube-scheduler設定ファイルの作成
```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clinetConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

- kube-scheduler systemdサービスファイルの作成
```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- Kubernetes Control Planeサービスの起動
```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler

sleep 10

# サービスの状態確認
sudo systemctl status kube-apiserver --no-pager
sudo systemctl status kube-controller-manager --no-pager
sudo systemctl status kube-scheduler --no-pager

---

# controller-0の出力
ubuntu@controller-0:~$ sudo systemctl daemon-reload
ubuntu@controller-0:~$ sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
Created symlink /etc/systemd/system/multi-user.target.wants/kube-apiserver.service → /etc/systemd/system/kube-apiserver.service.
Created symlink /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service → /etc/systemd/system/kube-controller-manager.service.
Created symlink /etc/systemd/system/multi-user.target.wants/kube-scheduler.service → /etc/systemd/system/kube-scheduler.service.
ubuntu@controller-0:~$ sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
ubuntu@controller-0:~$ sleep 10
ubuntu@controller-0:~$ sudo systemctl status kube-apiserver --no-pager
● kube-apiserver.service - Kubernetes API Server
     Loaded: loaded (/etc/systemd/system/kube-apiserver.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-10-22 22:43:11 UTC; 27s ago
       Docs: https://github.com/kubernetes/kubernetes
   Main PID: 17536 (kube-apiserver)
      Tasks: 9 (limit: 2218)
     Memory: 302.0M
        CPU: 3.310s
     CGroup: /system.slice/kube-apiserver.service
             └─17536 /usr/local/bin/kube-apiserver --advertise-address=192.168.8.10 --allow-privileged=true --apis…

Oct 22 22:43:16 controller-0 kube-apiserver[17536]: [-]poststarthook/rbac/bootstrap-roles failed: not finished
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: I1022 22:43:16.425237   17536 healthz.go:261] poststarth…readyz
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: [-]poststarthook/rbac/bootstrap-roles failed: not finished
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: I1022 22:43:16.492598   17536 controller.go:624] quota a…k8s.io
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: I1022 22:43:16.524474   17536 healthz.go:261] poststarth…readyz
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: [-]poststarthook/rbac/bootstrap-roles failed: not finished
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: I1022 22:43:16.639181   17536 alloc.go:330] "allocated c….0.1"}
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: W1022 22:43:16.674497   17536 lease.go:265] Resetting en….8.12]
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: I1022 22:43:16.675441   17536 controller.go:624] quota a…points
Oct 22 22:43:16 controller-0 kube-apiserver[17536]: I1022 22:43:16.702904   17536 controller.go:624] quota a…k8s.io
Hint: Some lines were ellipsized, use -l to show in full.
ubuntu@controller-0:~$ sudo systemctl status kube-controller-manager --no-pager
● kube-controller-manager.service - Kubernetes Controller Manager
     Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-10-22 22:43:11 UTC; 34s ago
       Docs: https://github.com/kubernetes/kubernetes
   Main PID: 17537 (kube-controller)
      Tasks: 5 (limit: 2218)
     Memory: 87.1M
        CPU: 531ms
     CGroup: /system.slice/kube-controller-manager.service
             └─17537 /usr/local/bin/kube-controller-manager --bind-address=0.0.0.0 --cluster-cidr=10.200.0.0/16 --…

Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: W1022 22:43:12.245345   17537 authentication.go:…work.
Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: W1022 22:43:12.245576   17537 authorization.go:1…work.
Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: I1022 22:43:12.248029   17537 controllermanager.…29.1"
Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: I1022 22:43:12.248191   17537 controllermanager.…CK=""
Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: I1022 22:43:12.249957   17537 tlsconfig.go:200] "Load…
Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: I1022 22:43:12.250459   17537 named_certificates.go:5…
Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: I1022 22:43:12.250644   17537 secure_serving.go:…10257
Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: I1022 22:43:12.250711   17537 tlsconfig.go:240] …ller"
Oct 22 22:43:12 controller-0 kube-controller-manager[17537]: I1022 22:43:12.253739   17537 leaderelection.go:…er...
Oct 22 22:43:13 controller-0 kube-controller-manager[17537]: E1022 22:43:13.856984   17537 leaderelection.go:332] …
Hint: Some lines were ellipsized, use -l to show in full.
ubuntu@controller-0:~$ sudo systemctl status kube-scheduler --no-pager
● kube-scheduler.service - Kubernetes Scheduler
     Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; vendor preset: enabled)
     Active: activating (auto-restart) (Result: exit-code) since Wed 2025-10-22 22:43:49 UTC; 194ms ago
       Docs: https://github.com/kubernetes/kubernetes
    Process: 17615 ExecStart=/usr/local/bin/kube-scheduler --config=/etc/kubernetes/config/kube-scheduler.yaml --v=2 (code=exited, status=1/FAILURE)
   Main PID: 17615 (code=exited, status=1/FAILURE)
        CPU: 300ms
ubuntu@controller-0:~$

```

controller-0とcontroller-2でkube-schedulerが動いていない。
```
ubuntu@controller-0:~$ sudo systemctl status kube-scheduler --no-pager
● kube-scheduler.service - Kubernetes Scheduler
     Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; vendor preset: enabled)
     Active: activating (auto-restart) (Result: exit-code) since Wed 2025-10-22 22:43:49 UTC; 194ms ago
       Docs: https://github.com/kubernetes/kubernetes
    Process: 17615 ExecStart=/usr/local/bin/kube-scheduler --config=/etc/kubernetes/config/kube-scheduler.yaml --v=2 (code=exited, status=1/FAILURE)
   Main PID: 17615 (code=exited, status=1/FAILURE)
        CPU: 300ms
ubuntu@controller-0:~$

---

ubuntu@controller-2:~$ sudo systemctl status kube-scheduler --no-pager
● kube-scheduler.service - Kubernetes Scheduler
     Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; vendor preset: enabled)
     Active: activating (auto-restart) (Result: exit-code) since Fri 2025-10-24 19:36:16 UTC; 1s ago
       Docs: https://github.com/kubernetes/kubernetes
    Process: 251006 ExecStart=/usr/local/bin/kube-scheduler --config=/etc/kubernetes/config/kube-scheduler.yaml --v=2 (code=exited, status=1/FAILURE)
   Main PID: 251006 (code=exited, status=1/FAILURE)
        CPU: 403ms
ubuntu@controller-2:~$

```

- エラーログ解析
controller-0の`jounalctl`を見る
```
sudo journalctl -u kube-scheduler -n 50 --no-pager

```

コマンドが違いそう
```
ubuntu@controller-0:~$ sudo journalctl -u kube-scheduler -n 50 --no-pager
[sudo] password for ubuntu:
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519476  251632 flags.go:64] FLAG: --authorization-webhook-cache-unauthorized-ttl="10s"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519479  251632 flags.go:64] FLAG: --bind-address="0.0.0.0"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519484  251632 flags.go:64] FLAG: --cert-dir=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519487  251632 flags.go:64] FLAG: --client-ca-file=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519491  251632 flags.go:64] FLAG: --config="/etc/kubernetes/config/kube-scheduler.yaml"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519494  251632 flags.go:64] FLAG: --contention-profiling="true"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519499  251632 flags.go:64] FLAG: --disabled-metrics="[]"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519503  251632 flags.go:64] FLAG: --feature-gates=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519507  251632 flags.go:64] FLAG: --help="false"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519510  251632 flags.go:64] FLAG: --http2-max-streams-per-connection="0"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519515  251632 flags.go:64] FLAG: --kube-api-burst="100"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519520  251632 flags.go:64] FLAG: --kube-api-content-type="application/vnd.kubernetes.protobuf"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519523  251632 flags.go:64] FLAG: --kube-api-qps="50"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519528  251632 flags.go:64] FLAG: --kubeconfig=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519532  251632 flags.go:64] FLAG: --leader-elect="true"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519535  251632 flags.go:64] FLAG: --leader-elect-lease-duration="15s"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519539  251632 flags.go:64] FLAG: --leader-elect-renew-deadline="10s"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519544  251632 flags.go:64] FLAG: --leader-elect-resource-lock="leases"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519548  251632 flags.go:64] FLAG: --leader-elect-resource-name="kube-scheduler"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519552  251632 flags.go:64] FLAG: --leader-elect-resource-namespace="kube-system"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519556  251632 flags.go:64] FLAG: --leader-elect-retry-period="2s"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519560  251632 flags.go:64] FLAG: --log-flush-frequency="5s"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519563  251632 flags.go:64] FLAG: --log-json-info-buffer-size="0"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519569  251632 flags.go:64] FLAG: --log-json-split-stream="false"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519572  251632 flags.go:64] FLAG: --logging-format="text"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519576  251632 flags.go:64] FLAG: --master=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519579  251632 flags.go:64] FLAG: --permit-address-sharing="false"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519582  251632 flags.go:64] FLAG: --permit-port-sharing="false"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519586  251632 flags.go:64] FLAG: --pod-max-in-unschedulable-pods-duration="5m0s"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519589  251632 flags.go:64] FLAG: --profiling="true"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519592  251632 flags.go:64] FLAG: --requestheader-allowed-names="[]"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519597  251632 flags.go:64] FLAG: --requestheader-client-ca-file=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519600  251632 flags.go:64] FLAG: --requestheader-extra-headers-prefix="[x-remote-extra-]"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519606  251632 flags.go:64] FLAG: --requestheader-group-headers="[x-remote-group]"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519611  251632 flags.go:64] FLAG: --requestheader-username-headers="[x-remote-user]"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519615  251632 flags.go:64] FLAG: --secure-port="10259"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519619  251632 flags.go:64] FLAG: --show-hidden-metrics-for-version=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519622  251632 flags.go:64] FLAG: --tls-cert-file=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519626  251632 flags.go:64] FLAG: --tls-cipher-suites="[]"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519630  251632 flags.go:64] FLAG: --tls-min-version=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519633  251632 flags.go:64] FLAG: --tls-private-key-file=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519637  251632 flags.go:64] FLAG: --tls-sni-cert-key="[]"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519641  251632 flags.go:64] FLAG: --v="2"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519646  251632 flags.go:64] FLAG: --version="false"
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519650  251632 flags.go:64] FLAG: --vmodule=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.519655  251632 flags.go:64] FLAG: --write-config-to=""
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: I1024 19:39:16.825090  251632 serving.go:380] Generated self-signed cert in-memory
Oct 24 19:39:16 controller-0 kube-scheduler[251632]: E1024 19:39:16.825427  251632 run.go:74] "command failed" err="strict decoding error: unknown field \"clinetConnection\""
Oct 24 19:39:16 controller-0 systemd[1]: kube-scheduler.service: Main process exited, code=exited, status=1/FAILURE
Oct 24 19:39:16 controller-0 systemd[1]: kube-scheduler.service: Failed with result 'exit-code'.
ubuntu@controller-0:~$

```

- 設定ファイルの確認
```
cat /etc/kubernetes/config/kube-scheduler.yaml
```

- 再度設定ファイルを作成
同期モードにしておく
```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

- 再起動
```
sudo systemctl restart kube-scheduler
sudo systemctl status kube-scheduler
```

全てactiveになった

- 動作確認
同期モードを解除しておく

```
# controller-0で確認

kubectl cluster-info --kubeconfig ~/admin.kubeconfig
# エラーの場合下記
sudo kubectl get componentstatuses --kubeconfig /var/lib/kubernetes/kube-controller-manager.kubeconfig

curl -k https://127.0.0.1:6443/version

---
ubuntu@controller-0:~$ sudo kubectl get componentstatuses --kubeconfig /var/lib/kubernetes/kube-controller-manager.kubeconfig
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
scheduler            Healthy   ok
etcd-0               Healthy   ok
controller-manager   Healthy   ok
ubuntu@controller-0:~$ curl -k https://127.0.0.1:6443/version
{
  "major": "1",
  "minor": "29",
  "gitVersion": "v1.29.1",
  "gitCommit": "bc401b91f2782410b3fb3f9acf43a995c4de90d2",
  "gitTreeState": "clean",
  "buildDate": "2024-01-17T15:41:12Z",
  "goVersion": "go1.21.6",
  "compiler": "gc",
  "platform": "linux/amd64"
}ubuntu@controller-0:~$

```

- RBAC for Kubelet Authorization
kubelete APIへのアクセうを許可するRBAC設定を作成 controller-0で実施
```
cat <<EOF | sudo kubectl apply --kubeconfig /var/lib/kubernetes/kube-controller-manager.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

---
Error from server (Forbidden): error when retrieving current configuration of:
Resource: "rbac.authorization.k8s.io/v1, Resource=clusterroles", GroupVersionKind: "rbac.authorization.k8s.io/v1, Kind=ClusterRole"
Name: "system:kube-apiserver-to-kubelet", Namespace: ""
from server for: "STDIN": clusterroles.rbac.authorization.k8s.io "system:kube-apiserver-to-kubelet" is forbidden: User "system:kube-controller-manager" cannot get resource "clusterroles" in API group "rbac.authorization.k8s.io" at the cluster scope
ubuntu@controller-0:~$

```

- admin.kubeconfigをcontroller-0に配布
gateway-01で実行
```
scp admin.kubeconfig ubuntu@192.168.8.10:~/
```

~/admin.kubeconfigを実行
```
cat <<EOF | kubectl apply --kubeconfig ~/admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF


---
ubuntu@controller-0:~$ cat <<EOF | kubectl apply --kubeconfig ~/admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
EOF   - "*"es/metrics
error: error validating "STDIN": error validating data: failed to download openapi: Get "https://192.168.3.18:6443/openapi/v2?timeout=32s": dial tcp 192.168.3.18:6443: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
ubuntu@controller-0:~$

```

gatewaty-01で実行
```
cd ~/k8s-certs

cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

---

ubuntu@gateway-01:~/k8s-certs$ cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -                        apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
error: error validating "STDIN": error validating data: failed to download openapi: Get "https://192.168.3.18:6443/openapi/v2?timeout=32s": dial tcp 192.168.3.18:6443: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
ubuntu@gateway-01:~/k8s-certs$

```

- HAProxy設定ファイルの作成
```
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
  log /dev/log local0
  log /dev/log local1 notice
  chroot /var/lib/haproxy
  stats socket /run/haproxy/admin.sock mode 660 level admin
  stats timeout 30s
  user haproxy
  group haproxy
  daemon

defaults
  log     global
  mode    http
  option  httplog
  option  dontlognull
  timeout connect 5000
  timeout client 50000
  timeout server 50000

frontend kubernetes-apiserver
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-apiserver

backend kubernetes-apiserver
  mode tcp
  option tcp-check
  balance roundrobin
  server controller-0 192.168.8.10:6443 check fall 3 rise 2
  server controller-1 192.168.8.11:6443 check fall 3 rise 2
  server controller-2 192.168.8.12:6443 check fall 3 rise 2
EOF
```

実行の確認
```
ubuntu@gateway-01:~/k8s-certs$ sudo systemctl restart haproxy
ubuntu@gateway-01:~/k8s-certs$ sudo systemctl enable haproxy
Synchronizing state of haproxy.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable haproxy
ubuntu@gateway-01:~/k8s-certs$ sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2025-10-24 20:25:21 UTC; 7s ago
       Docs: man:haproxy(1)
             file:/usr/share/doc/haproxy/configuration.txt.gz
   Main PID: 20213 (haproxy)
      Tasks: 2 (limit: 2220)
     Memory: 71.2M
        CPU: 66ms
     CGroup: /system.slice/haproxy.service
             ├─20213 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.>
             └─20215 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.>

Oct 24 20:25:21 gateway-01 haproxy[20110]: [WARNING]  (20110) : Exiting Master process...
Oct 24 20:25:21 gateway-01 haproxy[20110]: [NOTICE]   (20110) : haproxy version is 2.4.24-0ubuntu0.22.04.3
Oct 24 20:25:21 gateway-01 haproxy[20110]: [NOTICE]   (20110) : path to executable is /usr/sbin/haproxy
Oct 24 20:25:21 gateway-01 haproxy[20110]: [ALERT]    (20110) : Current worker #1 (20112) exited with code 143 (Te>
Oct 24 20:25:21 gateway-01 haproxy[20110]: [WARNING]  (20110) : All workers exited. Exiting... (0)
Oct 24 20:25:21 gateway-01 systemd[1]: haproxy.service: Deactivated successfully.
Oct 24 20:25:21 gateway-01 systemd[1]: Stopped HAProxy Load Balancer.
Oct 24 20:25:21 gateway-01 systemd[1]: Starting HAProxy Load Balancer...
Oct 24 20:25:21 gateway-01 haproxy[20213]: [NOTICE]   (20213) : New worker #1 (20215) forked
Oct 24 20:25:21 gateway-01 systemd[1]: Started HAProxy Load Balancer.
lines 1-23/23 (END)
ubuntu@gateway-01:~/k8s-certs$


```

ClusterRole設定できた
```
}ubuntu@gateway-01:~/k8s-certs$cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
ubuntu@gateway-01:~/k8s-certs$


```

ClusterRoleBindingの作成
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

リソース確認
```
ubuntu@gateway-01:~/k8s-certs$ kubectl get clusterrole system:kube-apiserver-to-kubelet --kubeconfig admin.kubeconfig
NAME                               CREATED AT
system:kube-apiserver-to-kubelet   2025-10-24T20:32:56Z
ubuntu@gateway-01:~/k8s-certs$ kubectl get clusterrolebinding system:kube-apiserver --kubeconfig admin.kubeconfig
NAME                    ROLE                                           AGE
system:kube-apiserver   ClusterRole/system:kube-apiserver-to-kubelet   48s
ubuntu@gateway-01:~/k8s-certs$

```

- gateway-01でのHTTPヘルスチェック設定
- CA証明書の配置
```
sudo mkdir -p /var/lib/kubernetes
sudo cp ca.pem /var/lib/kubernetes
```

- nginxのインストール
```
sudo apt update
sudo apt install -y nginx
```

- nginx設定ファイルの作成
```
cat <<EOF | sudo tee /etc/nginx/sites-available/kubernetes.default.svc.cluster.local
server {
  listen 80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
    proxy_pass https://127.0.0.1:6443/healthz;
    proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

- シンボリックリンクの作成
```
sudo ln -sf /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
```

- nginxの再起動
```
sudo nginx -t  # 設定テスト
sudo systemctl restart nginx
sudo systemctl enable nginx

---
ubuntu@gateway-01:~/k8s-certs$ sudo nginx -t  # 設定テスト
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
ubuntu@gateway-01:~/k8s-certs$ sudo systemctl restart nginx
ubuntu@gateway-01:~/k8s-certs$ sudo systemctl enable nginx
Synchronizing state of nginx.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable nginx
ubuntu@gateway-01:~/k8s-certs$

```

```
curl -H "Host: kubernetes.default.svc.cluster.local" http://127.0.0.1/healthz

---
ubuntu@gateway-01:~/k8s-certs$ curl -H "Host: kubernetes.default.svc.cluster.local" http://127.0.0.1/healthz
okubuntu@gateway-01:~/k8s-certs$

```

```
[Q] なぜ最初にcontroller-0で実行しようとしたのか？

- 本来はcontrollerノードで実行するのが正統派
- しかし、準備（証明書配布、kubeconfig作成）が不足していた
- 結果的にgateway-01から実行する方が簡単だった
```

## 9章 Bootstrapping the Kubernetes Worker Nodes
### 概要
このステップでは、3つのworkerノードに以下をインストールします：
1. 依存パッケージ（socat, conntrack, ipset）
2. containerd（コンテナランタイム）
3. CNIプラグイン（ネットワーク）
4. kubelet（Podの管理）
5. kube-proxy（ネットワークプロキシ）

- 証明書とkubeconfigの配布
```
#worker-0
scp cap.pem worker-0-key.pem worker-0.pem \
  worker-0.kubeconfig kube-proxy.kubeconfig \
  ubuntu@192.168.8.20:~/


#worker-1
scp cap.pem worker-1-key.pem worker-1.pem \
  worker-1.kubeconfig kube-proxy.kubeconfig \
  ubuntu@192.168.8.21:~/


#worker-2
scp cap.pem worker-2-key.pem worker-2.pem \
  worker-2.kubeconfig kube-proxy.kubeconfig \
  ubuntu@192.168.8.22:~/
```

workerノードを3つのペインに分割する
同期モードを有効化しておく

- 依存パッケージのインストール
```
sudo apt update
sudo apt install -y socat conntrack ipset

```

- swapの無効化確認
[q]どうしてswapを無効化するの
```
sudo swapon --show
free -h
```

- ディレクトリの作成
```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

ディレクトリの説明：
- `/etc/cni/net.d`: CNI設定ファイル
- `/opt/cni/bin`: CNIプラグインバイナリ
- `/var/lib/kubelet`: kubelet用データ
- `/var/lib/kube-proxy`: kube-proxy用データ

- バイナリのダウンロードとインストール
containerd、CNIプラグイン、kubelet、kube-proxyのダウンロード
```
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.29.0/crictl-v1.29.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz \
  https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-linux-amd64.tar.gz \
  https://dl.k8s.io/v1.29.1/bin/linux/amd64/kubectl \
  https://dl.k8s.io/v1.29.1/bin/linux/amd64/kube-proxy \
  https://dl.k8s.io/v1.29.1/bin/linux/amd64/kubelet

```


- ディレクトリの作成
- バイナリのインストール
```
sudo mkdir -p /etc/containerd

sudo mv runc.amd64 runc
chmod +x runc kubectl kube-proxy kubelet
sudo mv runc kubectl kube-proxy kubelet /usr/local/bin/

sudo tar -xvf cni-plugins-linux-amd64-v1.4.0.tgz -C /opt/cni/bin/
sudo tar -xvf containerd-1.7.13-linux-amd64.tar.gz -C /usr/local/aws-cli/

tar -xvf crictl-v1.29.0-linux-amd64.tar.gz
sudo mv crictl /usr/local/bin/
```

- CNIネットワークの設定
同期モードを解除しておく

- worker-0
```
POD_CIDR=10.200.0.0/24

cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cnio0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "${POD_CIDR}"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
EOF
```

- loopback設定（全worker共通）
```
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
  "cniVersion": "1.0.0",
  "name": "lo",
  "type": "loopback"
}
EOF

```

- worker-1での設定
```
POD_CIDR=10.200.1.0/24

cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cnio0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "${POD_CIDR}"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
EOF

```

- worker-2での設定
```
POD_CIDR=10.200.2.0/24

cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cnio0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "${POD_CIDR}"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
EOF

```

- containerdの設定
同期モードを有効化しておく

- containerd設定ファイルの作成
```
cat <<EOF | sudo tee /etc/containerd/config.toml
version = 2

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      snapshotter = "overlayfs"
      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
        runtime_type = "io.containerd.runc.v2"
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
          SystemdCgroup = true
EOF


```

- containerd systemdサービスファイルの作成
```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF


```

- kubeletの設定
同期モードを解除しておく

- worker-0での設定
```
sudo cp ~/worker-0-key.pem ~/worker-0.pem /var/lib/kubelet/
sudo cp ~/worker-0.kubeconfig /var/lib/kubelet/kubeconfig
sudo cp ~/ca.pem /var/lib/kubernetes/

# kubelet設定ファイルの作成
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "10.200.0.0/24"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/worker-0.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/worker-0-key.pem"
EOF

```

- worker-1での設定
```
sudo cp ~/worker-1-key.pem ~/worker-1.pem /var/lib/kubelet/
sudo cp ~/worker-1.kubeconfig /var/lib/kubelet/kubeconfig
sudo cp ~/ca.pem /var/lib/kubernetes/

# kubelet設定ファイルの作成
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "10.200.1.0/24"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/worker-1.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/worker-1-key.pem"
EOF

```

- worker-2での設定
```
sudo cp ~/worker-2-key.pem ~/worker-2.pem /var/lib/kubelet/
sudo cp ~/worker-2.kubeconfig /var/lib/kubelet/kubeconfig
sudo cp ~/ca.pem /var/lib/kubernetes/

# kubelet設定ファイルの作成
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "10.200.2.0/24"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/worker-2.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/worker-2-key.pem"
EOF

```

- kubelet systemdサービスファイルの作成
```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```

- kube-proxyの設定
- kube-proxy kubeconfigの配置
```
sudo cp ~/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

- kube-proxy設定ファイルの作成
```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF

```

設定の説明：
- `mode: "iptables"`: iptablesモードを使用（デフォルト）
- `clusterCIDR`: 全Podの IPレンジ（10.200.0.0/16）

- kube-proxy systemdサービスファイルの作成
```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- サービスの起動
```
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

- サービスの状態確認
```
sudo systemctl status containerd --no-pager
sudo systemctl status kubelet --no-pager
sudo systemctl status kube-proxy --no-pager

---
ubuntu@worker-0:~$ sudo systemctl status containerd --no-pager
● containerd.service - containerd container runtime
     Loaded: loaded (/etc/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2025-10-25 05:19:53 UTC; 2min 14s ago
       Docs: https://containerd.io
    Process: 20802 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 20807 (containerd)
      Tasks: 8 (limit: 4555)
     Memory: 16.5M
        CPU: 250ms
     CGroup: /system.slice/containerd.service
             └─20807 /usr/local/bin/containerd

Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.485675164Z" level=info msg="skipping …plugin"
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486020448Z" level=info msg="Start sub… event"
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486087043Z" level=info msg="Start rec… state"
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486163629Z" level=info msg="Start eve…onitor"
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486179639Z" level=info msg="Start sna…syncer"
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486187694Z" level=info msg="Start cni…efault"
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486196431Z" level=info msg="Start str…server"
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486435935Z" level=info msg=serving...…k.ttrpc
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486516688Z" level=info msg=serving...…rd.sock
Oct 25 05:19:53 worker-0 containerd[20807]: time="2025-10-25T05:19:53.486542227Z" level=info msg="container…56411s"
Hint: Some lines were ellipsized, use -l to show in full.
ubuntu@worker-0:~$ sudo systemctl status kubelet --no-pager
● kubelet.service - Kubernetes Kubelet
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
     Active: activating (auto-restart) (Result: exit-code) since Sat 2025-10-25 05:22:15 UTC; 1s ago
       Docs: https://github.com/kubernetes/kubernetes
    Process: 21179 ExecStart=/usr/local/bin/kubelet --config=/var/lib/kubelet/kubelet-config.yaml --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --kubeconfig=/var/lib/kubelet/kubeconfig --v=2 (code=exited, status=1/FAILURE)
   Main PID: 21179 (code=exited, status=1/FAILURE)
        CPU: 64ms
ubuntu@worker-0:~$ sudo systemctl status kube-proxy --no-pager
● kube-proxy.service - Kubernetes Kube Proxy
     Loaded: loaded (/etc/systemd/system/kube-proxy.service; enabled; vendor preset: enabled)
     Active: activating (auto-restart) (Result: exit-code) since Sat 2025-10-25 05:22:25 UTC; 3s ago
       Docs: https://github.com/kubernetes/kubernetes
    Process: 21208 ExecStart=/usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/kube-proxy-config.yaml (code=exited, status=1/FAILURE)
   Main PID: 21208 (code=exited, status=1/FAILURE)
        CPU: 37ms
ubuntu@worker-0:~$

```

エラーが発生したのでkubeletとkube-proxyのログをみる
```
sudo journalctl -u kubelet -n 50 --no-pager | tail -30
sudo journalctl -u kube-proxy -n 50 --no-pager | tail -30

---
ubuntu@worker-0:~$ sudo journalctl -u kubelet -n 50 --no-pager | tail -30
sudo journalctl -u kube-proxy -n 50 --no-pager | tail -30
[sudo] password for ubuntu:
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085002   30795 flags.go:64] FLAG: --tls-cert-file=""
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085005   30795 flags.go:64] FLAG: --tls-cipher-suites="[]"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085012   30795 flags.go:64] FLAG: --tls-min-version=""
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085015   30795 flags.go:64] FLAG: --tls-private-key-file=""
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085018   30795 flags.go:64] FLAG: --topology-manager-policy="none"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085021   30795 flags.go:64] FLAG: --topology-manager-policy-options=""
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085024   30795 flags.go:64] FLAG: --topology-manager-scope="container"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085027   30795 flags.go:64] FLAG: --v="2"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085032   30795 flags.go:64] FLAG: --version="false"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085036   30795 flags.go:64] FLAG: --vmodule=""
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085040   30795 flags.go:64] FLAG: --volume-plugin-dir="/usr/libexec/kubernetes/kubelet-plugins/volume/exec/"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085043   30795 flags.go:64] FLAG: --volume-stats-agg-period="1m0s"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.085097   30795 feature_gate.go:249] feature gates: &{map[]}
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.088506   30795 server.go:487] "Kubelet version" kubeletVersion="v1.29.1"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.088522   30795 server.go:489] "Golang settings" GOGC="" GOMAXPROCS="" GOTRACEBACK=""
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.088589   30795 feature_gate.go:249] feature gates: &{map[]}
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.088685   30795 feature_gate.go:249] feature gates: &{map[]}
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.089904   30795 dynamic_cafile_content.go:119] "Loaded a new CA Bundle and Verifier" name="client-ca-bundle::/var/lib/kubernetes/ca.pem"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.090791   30795 dynamic_cafile_content.go:157] "Starting controller" name="client-ca-bundle::/var/lib/kubernetes/ca.pem"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.092088   30795 remote_runtime.go:143] "Validated CRI v1 runtime API"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.093000   30795 remote_image.go:111] "Validated CRI v1 image API"
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.095752   30795 fs.go:132] Filesystem UUIDs: map[01c73b9f-512b-488e-965c-3b9cc5f22468:/dev/sda2 2024-09-11-18-46-48-00:/dev/sr0 8ba5edde-9760-41ea-9cca-70bd5b7e58ef:/dev/dm-0]
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.095775   30795 fs.go:133] Filesystem partitions: map[/dev/mapper/ubuntu--vg-ubuntu--lv:{mountpoint:/ major:253 minor:0 fsType:ext4 blockSize:0} /dev/sda2:{mountpoint:/boot major:8 minor:2 fsType:ext4 blockSize:0} /dev/shm:{mountpoint:/dev/shm major:0 minor:28 fsType:tmpfs blockSize:0} /run:{mountpoint:/run major:0 minor:26 fsType:tmpfs blockSize:0} /run/lock:{mountpoint:/run/lock major:0 minor:29 fsType:tmpfs blockSize:0} /run/snapd/ns:{mountpoint:/run/snapd/ns major:0 minor:26 fsType:tmpfs blockSize:0} /run/user/1000:{mountpoint:/run/user/1000 major:0 minor:46 fsType:tmpfs blockSize:0}]
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.097819   30795 manager.go:217] Machine: {Timestamp:2025-10-25 06:24:22.09768751 +0000 UTC m=+0.069251020 CPUVendorID:AuthenticAMD NumCores:2 NumPhysicalCores:2 NumSockets:1 CpuFrequency:1996247 MemoryCapacity:4101840896 SwapCapacity:1912598528 MemoryByType:map[] NVMInfo:{MemoryModeCapacity:0 AppDirectModeCapacity:0 AvgPowerBudget:0} HugePages:[{PageSize:1048576 NumPages:0} {PageSize:2048 NumPages:0}] MachineID:2b9eafe405794f6da1871399efa8b120 SystemUUID:2b9eafe4-0579-4f6d-a187-1399efa8b120 BootID:7e8de33c-4406-4e35-94de-f042ef4b624c Filesystems:[{Device:/dev/mapper/ubuntu--vg-ubuntu--lv DeviceMajor:253 DeviceMinor:0 Capacity:10464022528 Type:vfs Inodes:655360 HasInodes:true} {Device:/dev/shm DeviceMajor:0 DeviceMinor:28 Capacity:2050920448 Type:vfs Inodes:500713 HasInodes:true} {Device:/run/lock DeviceMajor:0 DeviceMinor:29 Capacity:5242880 Type:vfs Inodes:500713 HasInodes:true} {Device:/dev/sda2 DeviceMajor:8 DeviceMinor:2 Capacity:1833099264 Type:vfs Inodes:116160 HasInodes:true} {Device:/run/snapd/ns DeviceMajor:0 DeviceMinor:26 Capacity:410185728 Type:vfs Inodes:500713 HasInodes:true} {Device:/run/user/1000 DeviceMajor:0 DeviceMinor:46 Capacity:410181632 Type:vfs Inodes:100142 HasInodes:true} {Device:/run DeviceMajor:0 DeviceMinor:26 Capacity:410185728 Type:vfs Inodes:500713 HasInodes:true}] DiskMap:map[253:0:{Name:dm-0 Major:253 Minor:0 Size:10737418240 Scheduler:none} 8:0:{Name:sda Major:8 Minor:0 Size:21474836480 Scheduler:none}] NetworkDevices:[{Name:ens18 MacAddress:bc:24:11:8e:75:06 Speed:-1 Mtu:1500}] Topology:[{Id:0 Memory:4101840896 HugePages:[{PageSize:1048576 NumPages:0} {PageSize:2048 NumPages:0}] Cores:[{Id:0 Threads:[0] Caches:[{Id:0 Size:65536 Type:Data Level:1} {Id:0 Size:65536 Type:Instruction Level:1} {Id:0 Size:524288 Type:Unified Level:2}] UncoreCaches:[] SocketID:0} {Id:1 Threads:[1] Caches:[{Id:1 Size:65536 Type:Data Level:1} {Id:1 Size:65536 Type:Instruction Level:1} {Id:1 Size:524288 Type:Unified Level:2}] UncoreCaches:[] SocketID:0}] Caches:[{Id:0 Size:16777216 Type:Unified Level:3} {Id:1 Size:16777216 Type:Unified Level:3}] Distances:[10]}] CloudProvider:Unknown InstanceType:Unknown InstanceID:None}
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.097880   30795 manager_no_libpfm.go:29] cAdvisor is build without cgo and/or libpfm support. Perf event counters are not available.
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.097905   30795 manager.go:233] Version: {KernelVersion:5.15.0-157-generic ContainerOsVersion:Ubuntu 22.04.5 LTS DockerVersion: DockerAPIVersion: CadvisorVersion: CadvisorRevision:}
Oct 25 06:24:22 worker-0 kubelet[30795]: I1025 06:24:22.098066   30795 server.go:745] "--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /"
Oct 25 06:24:22 worker-0 kubelet[30795]: E1025 06:24:22.098220   30795 run.go:74] "command failed" err="failed to run Kubelet: running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename\t\t\t\tType\t\tSize\t\tUsed\t\tPriority /swap.img                               file\t\t1867772\t\t0\t\t-2]"
Oct 25 06:24:22 worker-0 systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
Oct 25 06:24:22 worker-0 systemd[1]: kubelet.service: Failed with result 'exit-code'.
Oct 25 06:24:01 worker-0 systemd[1]: kube-proxy.service: Main process exited, code=exited, status=1/FAILURE
Oct 25 06:24:01 worker-0 systemd[1]: kube-proxy.service: Failed with result 'exit-code'.
Oct 25 06:24:06 worker-0 systemd[1]: kube-proxy.service: Scheduled restart job, restart counter is at 734.
Oct 25 06:24:06 worker-0 systemd[1]: Stopped Kubernetes Kube Proxy.
Oct 25 06:24:06 worker-0 systemd[1]: Started Kubernetes Kube Proxy.
Oct 25 06:24:06 worker-0 kube-proxy[30752]: E1025 06:24:06.298494   30752 server.go:556] "Error running ProxyServer" err="stat /var/lib/kube-proxy/kubeconfig: no such file or directory"
Oct 25 06:24:06 worker-0 kube-proxy[30752]: E1025 06:24:06.299567   30752 run.go:74] "command failed" err="stat /var/lib/kube-proxy/kubeconfig: no such file or directory"
Oct 25 06:24:06 worker-0 systemd[1]: kube-proxy.service: Main process exited, code=exited, status=1/FAILURE
Oct 25 06:24:06 worker-0 systemd[1]: kube-proxy.service: Failed with result 'exit-code'.
Oct 25 06:24:11 worker-0 systemd[1]: kube-proxy.service: Scheduled restart job, restart counter is at 735.
Oct 25 06:24:11 worker-0 systemd[1]: Stopped Kubernetes Kube Proxy.
Oct 25 06:24:11 worker-0 systemd[1]: Started Kubernetes Kube Proxy.
Oct 25 06:24:11 worker-0 kube-proxy[30766]: E1025 06:24:11.552057   30766 server.go:556] "Error running ProxyServer" err="stat /var/lib/kube-proxy/kubeconfig: no such file or directory"
Oct 25 06:24:11 worker-0 kube-proxy[30766]: E1025 06:24:11.552522   30766 run.go:74] "command failed" err="stat /var/lib/kube-proxy/kubeconfig: no such file or directory"
Oct 25 06:24:11 worker-0 systemd[1]: kube-proxy.service: Main process exited, code=exited, status=1/FAILURE
Oct 25 06:24:11 worker-0 systemd[1]: kube-proxy.service: Failed with result 'exit-code'.
Oct 25 06:24:16 worker-0 systemd[1]: kube-proxy.service: Scheduled restart job, restart counter is at 736.
Oct 25 06:24:16 worker-0 systemd[1]: Stopped Kubernetes Kube Proxy.
Oct 25 06:24:16 worker-0 systemd[1]: Started Kubernetes Kube Proxy.
Oct 25 06:24:16 worker-0 kube-proxy[30780]: E1025 06:24:16.807890   30780 server.go:556] "Error running ProxyServer" err="stat /var/lib/kube-proxy/kubeconfig: no such file or directory"
Oct 25 06:24:16 worker-0 kube-proxy[30780]: E1025 06:24:16.808560   30780 run.go:74] "command failed" err="stat /var/lib/kube-proxy/kubeconfig: no such file or directory"
Oct 25 06:24:16 worker-0 systemd[1]: kube-proxy.service: Main process exited, code=exited, status=1/FAILURE
Oct 25 06:24:16 worker-0 systemd[1]: kube-proxy.service: Failed with result 'exit-code'.
Oct 25 06:24:22 worker-0 systemd[1]: kube-proxy.service: Scheduled restart job, restart counter is at 737.
Oct 25 06:24:22 worker-0 systemd[1]: Stopped Kubernetes Kube Proxy.
Oct 25 06:24:22 worker-0 systemd[1]: Started Kubernetes Kube Proxy.
Oct 25 06:24:22 worker-0 kube-proxy[30794]: E1025 06:24:22.052981   30794 server.go:556] "Error running ProxyServer" err="stat /var/lib/kube-proxy/kubeconfig: no such file or directory"
Oct 25 06:24:22 worker-0 kube-proxy[30794]: E1025 06:24:22.053426   30794 run.go:74] "command failed" err="stat /var/lib/kube-proxy/kubeconfig: no such file or directory"
Oct 25 06:24:22 worker-0 systemd[1]: kube-proxy.service: Main process exited, code=exited, status=1/FAILURE
Oct 25 06:24:22 worker-0 systemd[1]: kube-proxy.service: Failed with result 'exit-code'.
ubuntu@worker-0:~$

```

- 問題1: swapが有効になっている（kubelet）
- 問題2: kube-proxy kubeconfigが存在しない

### 解決方法
- swapを無効化
```
sudo swapoff -a

sudo sed -i '/swap/d' /etc/fstab

sudo rm -f /swap.img

free -h
sudo swapon --show
```

- kube-proxy kubeconfigの配置確認
```
ls -l ~/kube-proxy.kubeconfig

sudo cp ~/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
sudo ls -l /var/lib/kube-proxy/kubeconfig
```

- サービスの再起動
```
sudo systemctl restart kubelet kube-proxy

sudo systemctl status kubelet --no-pager
sudo systemctl status kube-proxy --no-pager
```

gateway-01でノードの登録を確認
```
kubectl get nodes --kubeconfig ~/k8s-certs/admin.kuberconfig

---
ubuntu@gateway-01:~/k8s-certs$ kubectl get nodes --kubeconfig ~/k8s-certs/admin.kubeconfig
NAME       STATUS   ROLES    AGE     VERSION
worker-0   Ready    <none>   5m3s    v1.29.1
worker-1   Ready    <none>   4m42s   v1.29.1
worker-2   Ready    <none>   4m42s   v1.29.1
ubuntu@gateway-01:~/k8s-certs$

```

## ステップ10: Configuring kubectl for Remote Access
### 概要
このステップでは、ローカルPCからKubernetesクラスターを操作できるようにkubectlを設定します。
現在、admin.kubeconfigはgateway-01にありますが、これを：
- ローカルPCにコピー
- kubectlで使用できるように設定

`kubectl`はローカルPCにインストールしている。

- 1. admin.kubeconfigをローカルPCにコピー
```
# ~/.kubeはすでに作成している

scp ubuntu@192.168.3.18:~/k8s-certs/admin.kubeconfig ~/.kube/config

chmod 600 ~/.kube/config
```

- 2. 接続確認
```
# クラスター情報の確認
kubectl cluster-info

# ノード一覧の確認
kubectl get nodes

# コンポーネントの状態確認
kubectl get componentstatuses

# 全リソースの確認
kubectl get all --all-namespaces

---
on ☸ admin on kubernetes-the-hard-way in default () memo-cho on  master [!?]
♥ ❯ kubectl cluster-info

Kubernetes control plane is running at https://192.168.3.18:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

on ☸ admin on kubernetes-the-hard-way in default () memo-cho on  master [!?]
♥ ❯ kubectl get nodes

NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   15m   v1.29.1
worker-1   Ready    <none>   15m   v1.29.1
worker-2   Ready    <none>   15m   v1.29.1

on ☸ admin on kubernetes-the-hard-way in default () memo-cho on  master [!?]
♥ ❯ kubectl get componentstatuses

Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
scheduler            Healthy   ok
etcd-0               Healthy   ok
controller-manager   Healthy   ok

on ☸ admin on kubernetes-the-hard-way in default () memo-cho on  master [!?]
♥ ❯ kubectl get all --all-namespaces

NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP   2d8h

on ☸ admin on kubernetes-the-hard-way in default () memo-cho on  master [!?]
♥
```

- 3.kubeconfigの内容確認
```
# 設定の確認
kubectl config view

# 現在のコンテキスト確認
kubectl config current-context

# クラスター一覧
kubectl config get-clusters

# ユーザー一覧
kubectl config get-users
```

## ステップ11: Provisioning Pod Network Routes
### 概要
現在、各workerノードのPodは自分のノード内でしか通信できません。異なるworkerノード間のPod通信を可能にするため、gateway-01にルーティング設定を追加します。

- Pod CIDRの割り当て

| Worker Node | Pod CIDR | IPレンジ |
| ----------- | -------- | -------- |
| worker-0    | 10.200.0.0/24 | 10.200.0.1 ~ 10.200.0.254 |
| worker-1    | 10.200.1.0/24 | 10.200.1.1 ~ 10.200.1.254 |
| worker-2    | 10.200.2.0/24 | 10.200.2.1 ~ 10.200.2.254 |


- gateway-01でのルーティング設定
- ルーティングテーブルの追加
```
# worker-0へのルート
sudo ip route add 10.200.0.0/24 via 192.168.8.20
# worker-1へのルート
sudo ip route add 10.200.1.0/24 via 192.168.8.21
# worker-2へのルート
sudo ip route add 10.200.2.0/24 via 192.168.8.22
```

- ルーティングテーブルの確認
```
ip route | grep "10.200"

---
ubuntu@gateway-01:~$ ip route | grep "10.200"
10.200.0.0/24 via 192.168.8.20 dev ens19
10.200.1.0/24 via 192.168.8.21 dev ens19
10.200.2.0/24 via 192.168.8.22 dev ens19
ubuntu@gateway-01:~$

```

- ルートの永続化
- Netplan設定ファイルの編集
```
sudo vim /etc/netplan/01-netcfg.yaml
```

cat結果
```
ubuntu@gateway-01:~$ cat /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
      dhcp4-overrides:
        route-metric: 100
    ens19:
      addresses:
        - 192.168.8.1/24
      dhcp4: false
      routes:
        - to: 10.200.0.0/24
          via: 192.168.8.20
        - to: 10.200.1.0/24
          via: 192.168.8.21
        - to: 10.200.2.0/24
          via: 192.168.8.22
ubuntu@gateway-01:~$

```

- 設定の適用、再度確認
```
sudo netplan apply
ip route | grep "10.200"

---
ubuntu@gateway-01:~$ sudo netplan apply

** (generate:22253): WARNING **: 10:46:01.329: Permissions for /etc/netplan/01-netcfg.yaml are too open. Netplan configuration should NOT be accessible by others.
WARNING:root:Cannot call Open vSwitch: ovsdb-server.service is not running.

** (process:22251): WARNING **: 10:46:01.647: Permissions for /etc/netplan/01-netcfg.yaml are too open. Netplan configuration should NOT be accessible by others.

** (process:22251): WARNING **: 10:46:01.775: Permissions for /etc/netplan/01-netcfg.yaml are too open. Netplan configuration should NOT be accessible by others.

** (process:22251): WARNING **: 10:46:01.775: Permissions for /etc/netplan/01-netcfg.yaml are too open. Netplan configuration should NOT be accessible by others.
ubuntu@gateway-01:~$ ip route | grep "10.200"
10.200.0.0/24 via 192.168.8.20 dev ens19 proto static
10.200.1.0/24 via 192.168.8.21 dev ens19 proto static
10.200.2.0/24 via 192.168.8.22 dev ens19 proto static
ubuntu@gateway-01:~$

```

- IP Forwardingの確認
```
# IP Forwardingが有効か確認
cat /proc/sys/net/ipv4/ip_forward

---
ubuntu@gateway-01:~$ cat /proc/sys/net/ipv4/ip_forward
1
ubuntu@gateway-01:~$

```

## ステップ12: Deploying the DNS Cluster Add-on
### 概要
CoreDNSは、Kubernetesクラスター内のDNSサーバーです。これにより：
- ServiceをDNS名で参照できる（例: `nginx.default.svc.cluster.local`）
- Pod間の名前解決が可能になる
- 外部DNSへのクエリも転送される

- CoreDNSマニフェストのダウンロードとデプロイ
ローカルPCで実施できるのでローカルPCで実施する
- CoreDNSマニフェストの作成
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
  - 
```

