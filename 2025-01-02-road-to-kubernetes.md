# road to kubernetes

**Date:** 2025-01-02 08:01:15

## chap1

- コンテナ技術の基本について書かれている
- Kubernetesの採用に関する課題
- 本書は以下を行なっていく
    - 手動デプロイ
    - コンテナ展開
    - Kubernetesデプロイ

## chap2

- PythonとNode.jsでサーバーサイドのアプリを作成する
- Pythonで下記のアプリをFastAPIを使って作成した

```py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_index():
    return {"Hello": "World"}


@app.get("/api/v1/hello-world/")
def read_hello_world():
    return {"what": "road", "where": "kubernetes", "version": "v1"}
```

- Express.jsで下記のアプリを作成した。

```js
const express = require('express');
const fs = require('fs');
const path = require('path')
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.send("<h1>Hello Express World!</h1>");
})

app.listen(port, () => {
  const appPid = path.resolve(__dirname, 'app.pid'); fs.writeFileSync(appPid, `${process.pid}`); console.log(`Server running on port http://127.0.0.1:${port}`);
})

app.get('/api/v2/rocket-man1/', (req, res) => {
  const myObject = { who: "rocker man", where: "mars" }; const jsonString = JSON.stringify(myObject); res.json(jsonString);
})
```

- Gitの使い方が一通り書いてある。
- GitHubの使い方が一通り書いてある。

## chap3

- Akamai Connected Cloudの上に作成したVMにDeployする。
- VMにbearリポジトリを作成して、リモートリポジトリとしてpushした。
- post-receive hookを作成して、pushされたら自動でデプロイするようにした。
```bash
#!/bin/bash
git --work-tree=$WORK_TREE --git-dir=$GIT_DIR checkout HEAD -f
```

- post-receive hookでrequirements.txtをインストールするようにした。
```bash
#!/bin/bash
git --work-tree=$WORK_TREE --git-dir=$GIT_DIR checkout HEAD -f

/opt/venv/bin/python -m pip install -r $WORK_TREE/src/requirements.txt
```

- post-receive hookでNode.jsの依存関係をインストールするようにした。
```bash
#!/bin/bash
git --work-tree=$WORK_TREE --git-dir=$GIT_DIR checkout HEAD -f
# Install the Node.js Requirements
cd $WORK_TREE
npm install
```

- Supervisorというツールを使って、プロセスを管理するようにする。
[https://supervisord.org/]

- jsを起動するSupervidorのconf
```conf
[program:roadtok8s-js]
direcotry=/opt/projects/roadtok8s/js/src
command=/root/.nvm/versions/node/v22.12.0/bin/node /opt/projects/roadtok8s/js/src/main.js
autostart=true
autorestart=true
startretries=3
stderr_logfile=/var/log/supervisor/roadtok8s/js/stderr.log
stdout_logfile=/var/log/supervisor/roadtok8s/js/stdout.log
```

- Pythonを起動するSupervidorのconf
```conf
[program:roadtok8s-py]
directory=/opt/projects/roadtok8s/py/src
command=/opt/venv/bin/gunicorn --worker-class uvicorn.workers.UvicornWorker main:app --bind "0.0.0.0:8888" --pid /var/run/roadtok8s-py.pid
autostart=true
autorestart=true
startretries=3
stderr_logfile=/var/log/supervisor/roadtok8s/py/stderr.log
stdout_logfile=/var/log/supervisor/roadtok8s/py/stdout.log
```

- post-receive hookでSupervisorを再起動するようにした。
```bash
#!/bin/bash
git --work-tree=/opt/projects/roadtok8s/py --git-dir=/var/repos/roadtok8s/py.git checkout HEAD -f

# Install the Python Requirements
/opt/venv/bin/python -m pip install -r /opt/projects/roadtok8s/py/src/requirements.txt

# Restart the Python Application in Supervisor
sudo supervisorctl restart roadtok8s-py

#!/bin/bash
git --work-tree=/opt/projects/roadtok8s/js --git-dir=/var/repos/roadtok8s/js.git checkout HEAD -f

# Install the Node.js Requirements
cd /opt/projects/roadtok8s/js
npm install

# Restart the Node.js Application in Supervisor
sudo supervisorctl restart roadtok8s-js
```

- nginxを使って、リバースプロキシを設定する。
- `/`のリクエストはPythonアプリに、`/js/`のリクエストはNode.jsアプリに振り分ける。
```conf
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://localhost:8888;
    }

    location /js/ {
        proxy_pass http://localhost:3000;
    }
}
```

- ufwを使って、ファイアウォールを設定する。
```
ufw allow ssh
ufw allow 'Nginx Full'
```

## chap4
Github Actionsを使ったデプロイ

- GitHub Actionsを使って、Ansibleを実行する。
```yaml
name: Run Ansible
on:
  workflow_dispatch:

jobs:
  run-playbooks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Python 3
        uses: actions/setup-python@v4
        with:
          python-version: "3.8"
      - name: Upgrade Pip & Install Ansible
        run: |
          python -m pip install --upgrade pip
          python -m pip install ansible
      - name: Implement the Private SSH Key
        run: |
          mkdir -p ~/.ssh/
          echo "@{{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
      - name: Ansible Inventory File for Remote host
        run: |
          mkdir -p ./devops/ansible/
          export INVENTORY_FILE=./devops/ansible/inventory.ini
          echo "[my_host_group]" > $INVENTORY_FILE
          echo "${{ secrets.AKAMAI_INSTANCE_IP_ADDRESS }}" >> $INVENTORY_FILE
      - name: Ansible Default Configuration File
        run: |
          mkdir -p ./devops/ansible/
          cat <<EOF > ./devops/ansible/ansible.cfg
          [defaults]
          ansible_python_interpreter = '/usr/bin/python3'
          ansible_ssh_private_key_file = ~/.ssh/id_rsa
          remote_user = root
          inventory = ./inventory.ini
          host_key_checking = False
          EOF
      - name: Ping Ansible Hosts
        working-directory: ./devops/ansible
        run: |
          ansible all -m ping
      - name: Run Ansible Playbooks
        working-directory: ./devops/ansible/
        run: |
          ansible-playbook install-nginx.yaml
      - name: Deploy Python via Ansible
        working-directory: ./devops/ansible/
        run: |
          ansible-playbook deploy-python.yaml
```
