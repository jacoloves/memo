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
