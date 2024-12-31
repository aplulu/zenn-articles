---
title: WSL2上でOllamaを使ってローカルLLMを推論実行する
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "windows", "wsl", "ollama", "llm" ]
published: true
---

WSL2上でOllamaを使ってローカルLLMを推論実行する方法を紹介します。

# はじめに

[Ollama](https://github.com/ollama/ollama)は、LLMを主にローカルで実行するためのOSSフレームワークです。
今回はOllamaによるLLMの実行環境をWSL2上に構築し、Docker上でOllamaとLLMを実行する方法を紹介します。

同時に[open-webui](https://github.com/open-webui/open-webui)を使用して、一般的なLLMサービスと同様にブラウザから気軽にLLMを実行できる環境を構築します。

おうちにゲーミングPCやNVIDIA GPUを搭載したPCがある方におすすめです。

# 前提環境

* Windows 10 or 11
* WSL2
* NVIDIA GPU (VRAM 16GB以上推奨です。)

# 実行手順

## WSL2のセットアップ

Microsoft StoreからUbuntu 22.04.05 LTSをインストールします。

https://apps.microsoft.com/detail/9pn20msr04dw?hl=ja-JP&gl=JP

## Dockerのインストール

以下の手順に従ってDockerをインストールします。
https://docs.docker.com/engine/install/ubuntu/

```shell
# Uninstall old versions
$ for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# Add Docker's official GPG key:
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update

# Install Docker
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker
$ sudo systemctl enable docker
$ sudo systemctl enable containerd
$ sudo systemctl start docker
```

一般ユーザでDockerを実行するために続けて、下記の手順に従って設定します。

https://docs.docker.com/engine/install/linux-postinstall/

```shell
# Create the docker group.
$ sudo groupadd docker

# Add your user to the docker group.
$ sudo usermod -aG docker $USER
$ newgrp docker
```

## NVIDIA Container Toolkitのインストール

次の手順に従ってNVIDIA Container Toolkitをインストールします。

https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

```shell
# Add the package repositories
$ curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Update the package repositories
$ sudo apt-get update

# Install the NVIDIA Container Toolkit
$ sudo apt-get install -y nvidia-container-toolkit

# Configure the runtime
$ sudo nvidia-ctk runtime configure --runtime=docker

# Restart Docker
$ sudo systemctl restart docker
```

ここまで完了すると、WSL2上のDockerからNVIDIA GPUを利用できるようになります。
以下のコマンドでGPUが利用できることを確認します。

```shell
$ docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

成功すると、以下のような出力が表示されます。

```
aplulu@ragdoll:~$ docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
Mon Dec 30 08:46:10 2024
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 545.36                 Driver Version: 546.33       CUDA Version: 12.3     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce RTX 3090        On  | 00000000:01:00.0  On |                  N/A |
| 44%   55C    P3              85W / 350W |   3297MiB / 24576MiB |     31%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+

+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|    0   N/A  N/A        25      G   /Xwayland                                 N/A      |
+---------------------------------------------------------------------------------------+
```

## Ollamaのセットアップ

適当なディレクトリにOllama用ボリュームなどを格納するディレクトリを作成し、Docker Composeの定義ファイルを作成します。

Ollama用のボリュームにはダウンロードしたモデルデータなどが保存されてるため、数十GB程度の容量が必要です。

```shell
$ mkdir -p ~/ollama
$ cd ~/ollama
$ cat <<EOF > compose.yaml
services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ./ollama:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
              driver: nvidia
              count: all
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "8080:8080"
    environment:
      API_URL: http://ollama:11434
      # 今回はローカルで実行するため、認証を無効化しています。
      WEBUI_AUTH: "False"
EOF
```

次のコマンドでOllamaとopen-webuiを起動します。

```shell
$ docker compose up -d
```

## モデルデータのダウンロード

Ollamaで推論を実行するためには、モデルデータが必要です。
今回はMicrosoftが開発したphi3モデルを使用します。
お使いのGPUのVRAMに合わせて以下のモデルデータをダウンロードしてください。

* 14Bモデル (推奨: VRAM 16GB以上)
  * https://ollama.com/library/phi3:14b-medium-4k-instruct-q6_K
* 3.8Bモデル (推奨: VRAM 8GB以上)
  * https://ollama.com/library/phi3:3.8b-mini-4k-instruct-q6_K

```shell
# 14Bモデルのダウンロード
$ docker compose exec ollama ollama pull phi3:14b-medium-4k-instruct-q6_K

# 3.8Bモデルのダウンロード
$ docker compose exec ollama ollama pull phi3:3.8b-mini-4k-instruct-q6_K
```

## 推論実行テスト

モデルのダウンロードが完了したら、curlで推論を実行してみます。

```shell
# 14Bモデルで推論を実行
$ curl -v http://localhost:11434/api/chat \
  -d '{"model":"phi3:14b-medium-4k-instruct-q6_K","stream":false,"messages":[{"role":"user","content":"What famous constellations are visible in December?"}]}'
# 3.8Bモデルで推論を実行
$ curl -v http://localhost:11434/api/chat \
  -d '{"model":"phi3:3.8b-mini-4k-instruct-q6_K","stream":false,"messages":[{"role":"user","content":"What famous constellations are visible in December?"}]}'
```

以下のようなレスポンスが返ってくれば成功です。初回はモデルのロードに時間がかかるため、数十秒程度かかることがあります。

```
aplulu@ragdoll:/mnt/d/ollama$ curl -v http://localhost:11434/api/chat -d '{"model":"phi3:14b-medium-4k-instruct-q6_K","stream":false,"messages":[{"role":"user","content":"What famous constellations are visible in December?"}]}'
*   Trying 127.0.0.1:11434...
* Connected to localhost (127.0.0.1) port 11434 (#0)
> POST /api/chat HTTP/1.1
> Host: localhost:11434
> User-Agent: curl/7.81.0
> Accept: */*
> Content-Length: 152
> Content-Type: application/x-www-form-urlencoded
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Content-Type: application/json; charset=utf-8
< Date: Tue, 31 Dec 2024 07:57:23 GMT
< Content-Length: 830
<
* Connection #0 to host localhost left intact
{"model":"phi3:14b-medium-4k-instruct-q6_K","created_at":"2024-12-31T07:57:23.905011899Z","message":{"role":"assistant","content":"In December, some of the most prominent constellations that can be seen from the Northern Hemisphere include Orion, with its distinctive three-star belt; Taurus, marked by the bright star Aldebaran and the Pleiades cluster (also known as the Seven Sisters); Gemini, featuring the twin stars Castor and Pollux; and Canis Major, home to Sirius, the brightest star in our night sky. In the Southern Hemisphere, constellations like Crux (the Southern Cross) are visible prominently during this time of year.\n\n-----"},"done_reason":"stop","done":true,"total_duration":4597126661,"load_duration":98263985,"prompt_eval_count":19,"prompt_eval_duration":7000000,"eval_count":131,"eval_duration":4490000000}
```

## open-webui

open-webuiを使用することで、ブラウザから気軽にLLMの推論を実行できるようになります。

http://localhost:8080 にアクセスして、色々と試してみてください。

![open-webui](/images/windows-wsl2-ollama/open-webui.png)

# 補足情報

## モデルデータのダウンロード

phi3以外にも様々なモデルデータが提供されています。

どのようなモデルデータが提供されているかは以下のURLから確認できます。

https://ollama.com/library

モデルデータをダウンロードする際は、`ollama pull`コマンドを使用します。

```shell
$ docker compose exec ollama ollama pull [モデル名]
```

## 簡易ベンチマーク

トークン出力速度を調べるには `--verbose` オプションを付けて実行することで表示することが出来ます。

```shell
$ docker compose exec ollama ollama run phi3:14b-medium-4k-instruct-q6_K --verbose "日本で12月に見える有名な星座で短いお話を執筆してください。"
```

https://www.youtube.com/watch?v=LjO50mdU0O8

わたしの環境のRTX 3090でphi3:14b-medium-4k-instruct-q6_Kモデルを使用した場合、47.91トークン/秒程度のトークン出力性能が得られました。

## LAN内の他のデバイスからアクセスする (Windows 10)

Windows 10の環境でLAN内の他のデバイスからOllamaやopen-webuiにアクセスするには、ポートプロキシーを設定する必要があります。

```shell
# Ollamaのポート11434を転送
netsh interface portproxy add v4tov4 listenport=11434 listenaddress=0.0.0.0 connectport=11434 connectaddress=(wsl hostname -I)
# open-webuiのポート8080を転送
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=(wsl hostname -I)
``` 

詳細については以下のURLを参照してください。
https://learn.microsoft.com/ja-jp/windows-server/networking/technologies/netsh/netsh-interface-portproxy

## LAN内の他のデバイスからアクセスする (WSL2のミラーモード Windows 11 22H2以降)

デフォルトではWSL2はNATモードで動作しているため、LAN内の他のデバイスからOllamaやopen-webuiにはアクセスすることができません。
WSL2から導入されたミラーモードを使用することで、LAN内の他のデバイスからアクセスすることができるようになります。

ミラーモードを有効にするには、ホームディレクトリに `.wslconfig` ファイルを作成し、以下のように記述します。

```shell
$ cd /mnt/c/Users/[ユーザ名]
$ cat <<EOF > .wslconfig
[wsl2]
networkingMode=mirrored 
EOF
```

`.wslconfig` ファイルを作成したら、WSL2を再起動します。以下のコマンドはPowerShellで実行します。

```shell
wsl --shutdown
```

続いてHyper-Vファイアウォールのルールを追加し、ポート11434と8080を開放します。
```shell
# Ollamaのポート11434を開放
New-NetFirewallHyperVRule -Name MyWebServer -DisplayName "Ollama" -Direction Inbound -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -Protocol TCP -LocalPorts 11434

# open-webuiのポート8080を開放
New-NetFirewallHyperVRule -Name MyWebServer -DisplayName "open-webui" -Direction Inbound -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -Protocol TCP -LocalPorts 8080
```

詳細は以下のURLを参照してください。

https://learn.microsoft.com/ja-jp/windows/security/operating-system-security/network-security/windows-firewall/hyper-v-firewall

# おわりに

ここまでで、ローカル環境でLLMを動かす準備が整いました！
open-webuiを使うことで、ブラウザから簡単にLLMと対話できますので、ぜひ色々と試してみてください。
