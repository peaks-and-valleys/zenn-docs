---
title: "ローカル LLM と GUI で対話して、ウェブ検索もできるようにする (Ollama / Open WebUI / SearXNG)"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [llm, ローカルllm, ollama, openwebui]
published: true
---

## これをやったモチベーション

生成AIの力にはあやかりたいが、持ち前の貧乏性のせいで Claude Pro, Max とかにお金を払うのがどうしても憚られる。あとプライバシーに関しても、Open AI や Claude のサービスを使うと多少気になるところはある。それを理由に使用をやめるほどではないとはいえ。

幸い手元に RTX 3070 搭載のマシンがあるので、これで動かせる範囲のモデルでいい感じのことができないか探っていた。とりあえず実用性のある LLM との対話環境は作れたと思うので、この記事を書くことにした。

## 使用ソフトウェア

### [Ollama](https://ollama.com)

ローカル LLM のモデルを管理したり実行したりするソフトウェア。CLI でお手軽にいろいろなモデルを試せて便利。GPU を積んでいないマシンだと実用可能な性能のモデルは動かせないかも。

### [Open WebUI](https://openwebui.com)

ローカル LLM と対話するための GUI を提供してくれる。CLI でも対話自体はできるのだが、後述のウェブ検索機能も欲しかったので導入した。

### [SearXNG](https://docs.searxng.org)

セルフホスティングのメタ検索エンジン。複数の検索エンジンにリクエストを投げて、結果をいい感じにまとめてくれる。正直 Open WebUI のドキュメントで名前を見るまで知らなかった。

## 環境構築

以下はすべて **WSL 2 (Ubuntu 24.04 on Windows 11)** で検証した。Docker はインストール済であることを前提にする。

### Ollama 

[ドキュメント](https://github.com/ollama/ollama/blob/main/docs/linux.md)  に従ってホスト OS にインストールする。Docker を使わなかったのは、これ以外のことにも使いたかったから。

Linux であれば、以下のコマンド一発で入る。

```sh
curl -fsSL https://ollama.com/install.sh | sh
```

Ollama は勝手に立ち上がって常駐していてほしいので、その設定を行う: [Adding Ollama as a startup service (recommended)](https://github.com/ollama/ollama/blob/main/docs/linux.md#adding-ollama-as-a-startup-service-recommended)

#### LLM モデル

https://www.ollama.com/library の中から、マシンスペックと相談しながら使いたいモデルを pull する。
記事執筆時点では `qwen3` がいい感じだったので、たとえば `ollama pull qwen3:8b` でインストールする。

### Open WebUI, SearNXG

この2つは Docker コンテナ上で動かすので、後述。

## `compose.yaml` の作成

先に完成形を示すと、こうなる。

```yaml: compose.yaml
name: open-webui-with-ollama
services:
  open-webui:
    container_name: open-webui
    image: ghcr.io/open-webui/open-webui:ollama
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities:
                - gpu
    network_mode: host
    volumes:
      - open-webui:/app/backend/data
    environment:
      - ENABLE_WEB_SEARCH=true
      - WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://127.0.0.1:8888/search?q=<query>
  searxng:
    container_name: searxng
    image: searxng/searxng:latest
    ports:
      - "8888:8080"
    volumes:
      - ./searxng:/etc/searxng:rw
    restart: unless-stopped
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "1"
volumes:
  open-webui:
    external: true
    name: open-webui
```

### Open WebUI 用のコンテナを定義

これも [Quick Start](https://docs.openwebui.com/getting-started/quick-start/) に従うだけ……だと思ったのだが、自分の環境ではホスト OS の Ollama との通信ができなかった。[ポートマッピングを使わない](https://docs.openwebui.com/troubleshooting/connection-error#-docker-connection-error)ことで回避した。

この段階での `compose.yaml` は以下。ウェブ検索が必要なければこれで十分動く。GPU サポートが不要であれば、`deploy` キーを削除して pull する image を `ghcr.io/open-webui/open-webui:main` にすれば問題ないはず。

```yaml: compose.yaml
name: open-webui-with-ollama
services:
  open-webui:
    container_name: open-webui
    image: ghcr.io/open-webui/open-webui:ollama
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities:
                - gpu
    network_mode: host
    volumes:
      - open-webui:/app/backend/data
volumes:
  open-webui:
    external: true
    name: open-webui
```

`docker compose up -d` -> `http://localhost:8080` にアクセスして、UI が表示されれば 🆗。初回アクセス時はユーザー名・パスワードの新規登録が求められる。
### SearXNG の導入

#### `./searxng/` ディレクトリの作成

この段階ではディレクトリの中身は空。後の手順で

#### Open WebUI の環境変数の設定

`ENABLE_WEB_SEARCH`, `WEB_SEARCH_ENGINE`, `SEARXNG_QUERY_URL` を定義する。

https://docs.openwebui.com/getting-started/env-configuration/#web-search

#### コンテナの定義

これも [Open WebUI のドキュメントにチュートリアルがあり](https://docs.openwebui.com/tutorials/web-search/searxng/#docker-compose-setup)、概ねそのまま。ポート番号被りを回避したくらい。おそらくこのドキュメントが書かれた後に環境変数の仕様が変わったため、こちらからのコピペだけでは動作しないことに注意。

再掲になるが、`compose.yaml` の最終系は以下。

```yaml: compose.yaml
name: open-webui-with-ollama
services:
  open-webui:
    container_name: open-webui
    image: ghcr.io/open-webui/open-webui:ollama
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities:
                - gpu
    network_mode: host
    volumes:
      - open-webui:/app/backend/data
    environment:
      - ENABLE_WEB_SEARCH=true
      - WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://127.0.0.1:8888/search?q=<query>
  searxng:
    container_name: searxng
    image: searxng/searxng:latest
    ports:
      - "8888:8080"
    volumes:
      - ./searxng:/etc/searxng:rw
    restart: unless-stopped
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "1"
volumes:
  open-webui:
    external: true
    name: open-webui
```

#### `./searxng/settings.yml` の修正

この段階で `docker compose up -d` すると、設定ファイル `./searxng/settings.yml` が生成される。ここに2点ほど、修正を加える。

- レスポンスを json で返せるようにする（[参考](https://github.com/open-webui/open-webui/discussions/3003#discussioncomment-9741643)）

```yaml
# ~lines 60
  # remove format to deny access, use lower case.
  # formats: [html, csv, json, rss]
  formats:
    - html
    - json
```

- 検索結果が返ってくるまでのタイムアウト時間を伸ばす（[参考](https://zenn.dev/link/comments/f1b49d76917709)）

```yaml
outgoing:
  # default timeout in seconds, can be override by engine
  request_timeout: 10.0
```

## 使用方法

1. `docker compose up -d`
2. `http://localhost:8080` にアクセス
	1. ついでに `http://localhost:8888` で SearXNG の検索も使える
