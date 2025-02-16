---
title: code-serverのリバースプロキシ環境下でのFlaskアプリ静的ファイルのパス問題
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "codeserver", "vscode", "flask" ]
published: true
---

# はじめに

[code-server](https://github.com/coder/code-server) は、VS CodeをWebブラウザ上で利用できるようにする便利なツールです。特に手元に開発環境が用意できない場合や、複数の環境から開発、チームでの共同作業で力を発揮します。

code-serverにはリバースプロキシ機能が含まれており、code-server上で動作させているアプリケーションに、code-server経由でアクセスすることができます。

https://coder.com/docs/code-server/guide#using-a-subpath

しかし、リバースプロキシ機能を利用してFlaskアプリケーションにアクセスする場合、パスの扱いに注意しないと、静的ファイルが正しく配信されず、404エラーが発生してしまうことがあります。

今回はcode-server上でFlaskアプリケーションを開発する際に遭遇する可能性のある、静的ファイル（CSS、JavaScript、画像など）の配信に関する問題と解決策について解説します。

# 問題：Flaskの静的ファイルパスとcode-server Proxyパスの不一致

Flaskでは、`url_for` 関数を使って静的ファイルへのURLを生成するのが一般的です。例えば、`static` フォルダ内の `style.css` にアクセスするには、テンプレート内で以下のように記述します。

```html
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
```

これにより通常、 `url_for` 関数は `/static/style.css` のようなURLを生成します。しかし、code-serverのリバースプロキシ機能を使ってFlaskアプリケーションにアクセスする場合、URLのパス部分は `/proxy/{{port}}/` から始まる形になります（`{{port}}` はFlaskアプリケーションが使用するポート番号）。

Flaskの `static_url_path` を適切に設定しないと、`url_for` が生成するURLに `/proxy/{{port}}/` が含まれず、静的ファイルが見つからないという問題が発生します。

`static_url_path` に `/proxy/{{port}}/static` を設定することでFlask側では正しいURLが生成されるようになりますが、これ単体では以下のような問題があります。

1. **Flask側のルーターとパスの不一致** Flaskは `/proxy/{{port}}/static` を静的ファイルのルートと認識しますが、code-serverのリバースプロキシはリクエストを転送する際に `/proxy/{{port}}` を削除します。つまり、Flaskには `/static` としてリクエストが届くため、対応するRouteがなく404エラーが発生します。
2. **`/absproxy/{{port}}` の利用**  code-serverには `/absproxy/{{port}}` という別のリバースプロキシ用のRouteも存在します。こちらを使うとリバースプロキシのパス部分は削除されないため問題は回避できますが、静的ファイル以外の全てのRouteにも `@app.route('/absproxy/{{port}}/api/hello')` のようにベースパスを付ける必要が出てきてしまい、コードが冗長になる上code-server以外の環境での動作が困難になります。

https://coder.com/docs/code-server/guide#prefixing-absproxyport-with-a-path

# 解決策：Middlewareでリクエストパスを書き換える

上記の問題を解決する一つの方法として、Middlewareを使ってリクエストパスを適切に書き換える方法を紹介します。

## 1. 環境変数の取得とProxyパスの生成

まず、code-serverが自動的に設定する環境変数 `VSCODE_PROXY_URI` からリバースプロキシのURIを取得し、Flaskのポート番号を置換してリバースプロキシのパスを生成します。

また、`static_url_path` に設定するパスもここで生成します。

```python
import os
from urllib.parse import urlparse
from flask import Flask, render_template

# Static directory
static_dir = 'static'

# Default Flask port
flask_port = 5000

# Use the absolute proxy path. If True, Application URL will be /absproxy/{{port}}/
use_absproxy = False

# Get the proxy URI from the environment variable
proxy_uri = os.getenv('VSCODE_PROXY_URI', '')

if use_absproxy:
    proxy_uri = proxy_uri.replace('/proxy', '/absproxy')

proxy_uri = proxy_uri.replace('{{port}}', str(flask_port))
proxy_path = urlparse(proxy_uri).path.rstrip('/') if proxy_uri else ''  # Proxyのパス部分(/proxy/{{port}} など)
static_path = f"{proxy_path}/{static_dir}" if proxy_path else f'/{static_dir}' # Flaskに設定するstatic_url_path

app = Flask(__name__, static_url_path=static_path, static_folder=static_dir)
```

## 2. Middlewareの作成

次に、リクエストパスを書き換えるMiddlewareを作成します。このMiddlewareは、以下の処理を行います。

* `/absproxy/{{port}}` を使用する場合、リクエストパスからこのプレフィックスを削除します。
* リクエストが静的ファイルに対するものである場合（`/static` で始まる場合）、`PATH_INFO` を `proxy_path` を付与した形に書き換えます。

```python
if proxy_uri:
    class FixProxyPathMiddleware():
        def __init__(self, app: Flask, static_dir, proxy_path):
            self.app = app
            self.static_dir = static_dir
            self.proxy_path = proxy_path

        def __call__(self, environ, start_response):
            if use_absproxy:
                environ['PATH_INFO'] = environ['PATH_INFO'].removeprefix(self.proxy_path)

            # If the request is for the static directory, rewrite the path to include the proxy path
            if environ['PATH_INFO'].startswith(f'/{self.static_dir}'):
                environ['PATH_INFO'] = self.proxy_path + environ['PATH_INFO']

            return self.app(environ, start_response)

    app.wsgi_app = FixProxyPathMiddleware(app.wsgi_app, static_dir, proxy_path)
```

## 3. Flaskアプリケーションの設定

最後に、Flaskアプリケーションの `static_url_path` と `static_folder` を設定し、作成したMiddlewareを適用します。

```python
@app.route("/")
def home():
    return render_template("index.html")

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=flask_port)
```

# サンプルプロジェクト

上記の手順を適用したサンプルプロジェクトをGitHubで公開しています。

https://github.com/aplulu/sample-code-server-flask-proxy-static-files

# まとめ

この記事では、code-server上でFlaskアプリケーションを開発する際に、静的ファイルの配信で発生する問題とその解決策について解説しました。

Middlewareを使ってリクエストパスを適切に書き換えることで、code-serverのリバースプロキシ環境下でもFlaskアプリケーションをスムーズに動作させることができます。

code-server上でのFlaskアプリケーションの開発で、静的ファイルのパス問題に遭遇した際にこの記事が参考になれば幸いです。

