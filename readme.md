# Nutanixの上でAIワークロードを動かすちょっとしたデモ

もともとこれはプリセールスSEさんにデモしてもらうために社内向けに作ったメモだったのですが、別に隠す必要もないので公開してしまいます。そういう経緯なので、Nutanix Cloud Infrastructure自体の構築手順は含んでいません。

GPUノードを持ったNutanixクラスタが既に動いていることを前提としています。GPUノードがなくてもできないことはないですが、その場合NotebookにあるPythonのライブラリや使用するモデルは多少変更する必要があります。

## クラスタのセットアップ

作業端末に必要なもの(Jump Hostを立てたほうがいいかも)

- kubectl https://kubernetes.io/docs/tasks/tools/
- helm https://helm.sh/docs/intro/install/
- kustomize https://github.com/kubernetes-sigs/kustomize
- git
- VSCodeとか

### Prism Centralのデプロイ

NKEとかObjectsとか動かすので、とりあえずX-Large(14vCPU/60GB)のSingle VMでデプロイしておきます。

### もろもろアップデート

新しめのサービスを動かすので、とりあえずLCMでもろもろ最新に上げといたほうが良さそうです。

2023/12/5時点：

- AOS: 6.5.4.5
- PC: pc.2023.3.01
- NKE: 2.9.0

死ぬほど時間かかりますのでギターの練習をしながら待ちましょう。


## 仮想マシン編

ほんとはKubernetesでやりたいところですが、それなりにハードルが高いのと、仮想マシンであればVMイメージ展開するだけで動かせますので、手っ取り早く済ませたい方はこちらで。（下記セットアップを終えた状態のOVAを取ってありますので、必要な方はご連絡ください。LLMとかも含んでいるので50GBぐらいありますが・・・）

勝手知ったるUbuntu22.04を使います。PassthroughでGPUを割り当てます。8vCPU/32GBぐらいでいいんじゃないでしょうか。

以下、作成したVM上での作業です。

### NVIDIA CUDA Toolkitを入れる

https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html
に書いてある通りです。

```
$ history
    1  gcc --version
    2  sudo apt install gcc
    3  sudo apt update
    4  sudo apt upgrade
    5  wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
    6  sudo dpkg -i cuda-keyring_1.1-1_all.deb
    7  sudo apt-get update
    8  sudo apt-get -y install cuda-toolkit-12-3
    9  sudo apt-get install -y cuda-drivers
```

GPUが見えているか確認します。

```
$ nvidia-smi
Wed Dec  6 10:09:53 2023
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 545.23.08              Driver Version: 545.23.08    CUDA Version: 12.3     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  Tesla T4                       Off | 00000000:00:06.0 Off |                  Off |
| N/A   51C    P0              26W /  70W |      2MiB / 16384MiB |      7%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+

+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|  No running processes found                                                           |
+---------------------------------------------------------------------------------------+
```

### JupyterHubのインストール

ターミナルでPythonプロンプトを叩くだけでもよいのですが、それだとデモ映えしないですし、マルチユーザーが使うプラットフォーム感を出したいので、JupyterHub/JupyterLabを入れることにします。

```
$ sudo apt install npm
$ sudo apt install pip
$ sudo pip3 install jupyterhub
$ sudo npm install -g configurable-http-proxy
```

JupyterHubにログインするユーザーを作ります

```
$ sudo adduser user1
$ su user1
$ mkdir ~/notebook
```

JupyterHubの設定をします。JupyterHubはsystemdのサービスとして起動する前提です。

```
$ sudo su
# mkdir /etc/jupyterhub
# cd /etc/jupyterhub
# jupyterhub --generate-config
```

`jupyterhub_config.py` とう設定ファイルが生成されるので、以下を書いておきます。

```/etc/jupyterhub/jupyterhub_config.py
#管理ユーザを追加
c.JupyterHub.admin_users = set(["admin"])
c.Authenticator.admin_users = set(["admin"]) 
c.JupyterHub.confirm_no_ssl = True #ssl通信を使わない
#jupyterを使用するユーザーを追加
c.Authenticator.allowed_users = set(["user1"])
#各ユーザのホームディレクトリのnotebook以下をjupyterが使用するように設定．デフォルトだとホームディレクトリが使用される．
c.Spawner.notebook_dir = '~/notebook'
```

JupyterHubをsystemdから起動するように設定します。

```/etc/systemd/system/jupyterhub.service
[Unit]
Description=Jupyterhub
After=syslog.target network.target

[Service]
User=root
ExecStart=/usr/local/bin/jupyterhub -f /etc/jupyterhub/jupyterhub_config.py

[Install]
WantedBy=multi-user.target
```

```
$ sudo systemctl start jupyterhub
$ sudo systemctl enable jupyterhub
```

サービスが起動したら、`http://<仮想マシンのIPアドレス>:8000/` でアクセスできます。先ほど作った`user1`のユーザー名とパスワードでログインするとJupyter Notebookが立ち上がります。

### Jupyter NotebookでLLMを遊ぶ

[./notebooks/](./notebooks/) 以下のサンプルコードで遊んでください。何をやっているかは見ればだいたいわかると思います。


## Kubernetes/KubeFlow編

### NKEのKubernetesクラスタをデプロイ

GPUノードを有効にするには、まずKubernetesクラスタをデプロイして、後からnode poolを追加するようです。
https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Engine-v2_9:top-gpu-configure-ui-t.html

そのため、まずは最小構成でk8sクラスタをデプロイします。とはいえCPUノードも3つぐらい欲しいので

- Control Plane: 1
- etcd: 1
- Worker: 3

としました。ドキュメント読まなくても雰囲気でできます。いちおうDefault Storage Class用のストレージコンテナは作っておいたほうがよいでしょう。

k8sクラスタができたら、Kubeconfigをダウンロードして、作業端末のkubectlが参照するようにします。操作するKubernetes環境が他にない場合は、 `~/.kube/config` というファイル名で保存すればOKです。

クラスタの状態を確認します。

```
$ kubectl get node
NAME                              STATUS   ROLES                  AGE   VERSION
k8s-gpu-cluster-bc300b-master-0   Ready    control-plane,master   16m   v1.26.8
k8s-gpu-cluster-bc300b-worker-0   Ready    node                   14m   v1.26.8
k8s-gpu-cluster-bc300b-worker-1   Ready    node                   14m   v1.26.8
k8s-gpu-cluster-bc300b-worker-2   Ready    node                   14m   v1.26.8
```

次にGPUノードをnode poolとして追加します。これはPrism CentralのKubernetes Managementからできます。GPUのアサインも含めて、特に迷うところはないと思います。

```
$ kubectl get node
NAME                                       STATUS   ROLES                  AGE     VERSION
k8s-gpu-cluster-bc300b-gpu-pool-worker-0   Ready    node                   4m20s   v1.26.8
k8s-gpu-cluster-bc300b-gpu-pool-worker-1   Ready    node                   4m21s   v1.26.8
k8s-gpu-cluster-bc300b-gpu-pool-worker-2   Ready    node                   4m20s   v1.26.8
k8s-gpu-cluster-bc300b-master-0            Ready    control-plane,master   25m     v1.26.8
k8s-gpu-cluster-bc300b-worker-0            Ready    node                   24m     v1.26.8
k8s-gpu-cluster-bc300b-worker-1            Ready    node                   24m     v1.26.8
k8s-gpu-cluster-bc300b-worker-2            Ready    node                   24m     v1.26.8
```

GPUを使えるようにするため、NVIDIA GPU Operatorをインストールします。helmで一発ですが、`--set toolkit.version=v1.14.3-centos7` を指定しないとうまく動きませんでした。

```
$ helm repo add nvidia https://nvidia.github.io/gpu-operator \
   && helm repo update

$ helm install --wait -n gpu-operator --create-namespace gpu-operator \
  nvidia/gpu-operator --set toolkit.version=v1.14.3-centos7
```

daemonsetが動くまでしばらくかかります

```
$ kubectl -n gpu-operator get pod
NAME                                                          READY   STATUS      RESTARTS   AGE
gpu-feature-discovery-54lr5                                   1/1     Running     0          4m47s
gpu-feature-discovery-9f8pf                                   1/1     Running     0          4m47s
gpu-feature-discovery-l64lv                                   1/1     Running     0          4m47s
gpu-operator-6fdbc66bd4-828ls                                 1/1     Running     0          5m16s
gpu-operator-node-feature-discovery-gc-7c8b8d65fd-9l78r       1/1     Running     0          5m16s
gpu-operator-node-feature-discovery-master-56874d94b9-4z9gc   1/1     Running     0          5m16s
gpu-operator-node-feature-discovery-worker-2s5hc              1/1     Running     0          5m16s
gpu-operator-node-feature-discovery-worker-8z46h              1/1     Running     0          5m16s
gpu-operator-node-feature-discovery-worker-d8g65              1/1     Running     0          5m16s
gpu-operator-node-feature-discovery-worker-dz6wl              1/1     Running     0          5m16s
gpu-operator-node-feature-discovery-worker-jkls8              1/1     Running     0          5m16s
gpu-operator-node-feature-discovery-worker-kvjd6              1/1     Running     0          5m16s
nvidia-container-toolkit-daemonset-9g9gw                      1/1     Running     0          4m47s
nvidia-container-toolkit-daemonset-b6hn8                      1/1     Running     0          4m47s
nvidia-container-toolkit-daemonset-qpkxn                      1/1     Running     0          4m47s
nvidia-cuda-validator-62x78                                   0/1     Completed   0          2m20s
nvidia-cuda-validator-d2k5q                                   0/1     Completed   0          2m17s
nvidia-cuda-validator-hnc89                                   0/1     Completed   0          2m11s
nvidia-dcgm-exporter-b77zz                                    1/1     Running     0          4m47s
nvidia-dcgm-exporter-jpsxx                                    1/1     Running     0          4m46s
nvidia-dcgm-exporter-wx7vm                                    1/1     Running     0          4m47s
nvidia-device-plugin-daemonset-4r4wq                          1/1     Running     0          4m47s
nvidia-device-plugin-daemonset-lmtcl                          1/1     Running     0          4m47s
nvidia-device-plugin-daemonset-wplks                          1/1     Running     0          4m47s
nvidia-driver-daemonset-7g48l                                 1/1     Running     0          4m54s
nvidia-driver-daemonset-g2jg6                                 1/1     Running     0          4m54s
nvidia-driver-daemonset-nxlr6                                 1/1     Running     0          4m54s
nvidia-operator-validator-d66zm                               1/1     Running     0          4m47s
nvidia-operator-validator-kmlrd                               1/1     Running     0          4m47s
nvidia-operator-validator-qfrrk                               1/1     Running     0          4m47s
```

`$ kubectl describe node k8s-gpu-cluster-bc300b-gpu-pool-worker-0` とかやって

```
Capacity:
  cpu:                8
  ephemeral-storage:  309504832Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32744112Ki
  nvidia.com/gpu:     1
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  309504832Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32334512Ki
  nvidia.com/gpu:     1
  pods:               110
```

と、`nvidia.com/gpu` が出ていれば準備完了です。


### Object Storeの作成

先にObject Storeを作っておくと、次の項でKubeFlowをデプロイするときにそれを使うように設定できます。なのでObjectsを有効にしててきとうに作っておきます。デモ用途ならSingle-Single構成で十分です。ユーザーも追加してAccess KeyとSecret Keyを保存しておきます。

### KubeFlowのデプロイ

これの通りにやっていきます https://nutanix.github.io/kubeflow-manifests/docs/install-kubeflow/#installing-kubeflow-with-nutanix-object-store

私はkustomizeは5.2.1使ってますが特に問題ないです。

KubeFlowはdockerhubから引っ張ってくるコンテナイメージの数が多いので、かなりの確率でDockerHubのrate limitに引っかかります。`kubectl -n kubeflow get pod` と打ってみて、`ImagePullBackOff` が複数出ていたらそれです。そうなると6時間とかかかるので、そうならないように、 `kubeflow-manifests/kubeflow/overlays/docker/docker-credentials.env` にDockerHubのアカウントを書いてから `bash install.sh -d` を実行すると良いです。くわしくは[こちら](https://github.com/nutanix/kubeflow-manifests/pull/17)

ユーザーアカウント作るところはすっとばしても良いです。デフォルトのuser / passwordは `user@example.com / 12341234` です。

### KubeFlowにアクセス

作業端末で

```
$ kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

として、ブラウザで `http://localhost:8080` を開くとKubeFlow Dashboardのログイン画面です。

ログインしたら、Notebooks で New notebookを作成します。

デフォルトで選択できるImageではCUDAのバージョンとの不整合があってうまく動かなかったので、Advanced Options → Custome Imageで `kubeflownotebookswg/jupyter-pytorch-cuda-full:latest` を指定しました。

8CPU、16GB、1GPUぐらい割り当てましょう。Workspace VolumeとData Volumeはそれぞれ50GBぐらいあったほうがいいと思います。

Notebookが上がってきたら `CONNECT` で開いて、このrepoの [./notebooks/](./notebooks/)以下にある.ipynbファイルをアップロードして遊んでください。




