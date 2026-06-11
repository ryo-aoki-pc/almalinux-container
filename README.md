# almalinux-container

blueprint(osbuild Image Builder の TOML 設定ファイル)を使って、カスタム AlmaLinux 10
コンテナイメージをビルドするリポジトリです。

## blueprint とは

blueprint は [osbuild Image Builder](https://osbuild.org/) が使用する TOML 形式の宣言的な
設定ファイルで、イメージに含めるパッケージやカスタマイズを記述します。

- **content セクション**: `[[packages]]`(パッケージ)、`[[groups]]`(パッケージグループ)、
  `[[containers]]`(埋め込みコンテナイメージ)
- **customizations セクション**: ユーザー、ファイル、ディレクトリ、サービス、ホスト名、
  ロケール、タイムゾーン、ファイアウォールなど

`container` イメージタイプでは、ファイルシステム/パーティション関連のカスタマイズは
使用できません(コンテナイメージにはパーティションが存在しないため)。

詳細は [Blueprint Reference](https://osbuild.org/docs/user-guide/blueprint-reference/) を
参照してください。

## リポジトリ構成

| ファイル | 説明 |
|---|---|
| `blueprint.toml` | コンテナイメージの定義(パッケージ・カスタマイズ) |
| `.github/workflows/build-container.yml` | GitHub Actions による自動ビルド |

## ビルド方法

### 方法A: image-builder-cli(推奨)

[image-builder-cli](https://github.com/osbuild/image-builder-cli) をコンテナとして実行する
方法です。ホストへのインストールが不要で、CI にも組み込みやすい方法です。

```bash
mkdir -p output
sudo podman run --rm --privileged \
  -v "$PWD/output:/output" \
  -v "$PWD/blueprint.toml:/blueprint.toml:ro" \
  ghcr.io/osbuild/image-builder-cli:latest \
  build container --distro almalinux-10.1 --blueprint /blueprint.toml
```

ビルドが完了すると `output/almalinux-10.1-container-x86_64/almalinux-10.1-container-x86_64.tar`
(OCI アーカイブ)が生成されます。

利用可能なディストリビューションとイメージタイプの一覧は次のコマンドで確認できます。

```bash
sudo podman run --rm ghcr.io/osbuild/image-builder-cli:latest list
```

### 方法B: osbuild-composer + composer-cli(AlmaLinux ホスト上)

AlmaLinux マシン上で従来の osbuild-composer サービスを使う方法です。

```bash
# インストールとサービス起動
sudo dnf install -y osbuild-composer composer-cli
sudo systemctl enable --now osbuild-composer.socket

# blueprint を登録してビルド開始
sudo composer-cli blueprints push blueprint.toml
sudo composer-cli blueprints list
sudo composer-cli compose types        # container が含まれることを確認
sudo composer-cli compose start almalinux-custom-container container

# 進捗確認(FINISHED になるまで待つ)
sudo composer-cli compose status

# 成果物(<UUID>-container.tar)のダウンロード
sudo composer-cli compose image <UUID>
```

## ビルドしたイメージの利用

`container.tar` は OCI アーカイブ形式です。skopeo または podman で取り込めます。

```bash
# skopeo でローカルのコンテナストレージへ取り込み(タグ付き)
sudo skopeo copy oci-archive:output/almalinux-10.1-container-x86_64/almalinux-10.1-container-x86_64.tar \
  containers-storage:localhost/almalinux-custom:latest

# 実行確認
sudo podman run --rm localhost/almalinux-custom:latest cat /etc/os-release
sudo podman run --rm -it localhost/almalinux-custom:latest tmux
```

`podman load -i <ビルドした .tar>` でも取り込み可能です(タグは付与されません)。

## CI(GitHub Actions)

`.github/workflows/build-container.yml` が main への push、pull request、手動実行
(workflow_dispatch)をトリガーに以下を行います。

1. image-builder-cli コンテナで `blueprint.toml` から AlmaLinux 10.1 の
   コンテナイメージをビルド
2. スモークテスト: イメージを取り込み、`/etc/os-release` が AlmaLinux であることを確認
3. lint: dockle でイメージがベストプラクティスに従っているか検査(WARN 以上で失敗)
4. ビルドした `.tar`(OCI アーカイブ)をワークフローアーティファクトとしてアップロード
5. GHCR(GitHub Container Registry)へ push(pull request 時は実行しない)

### イメージの lint(dockle)

ビルドしたイメージは [dockle](https://github.com/goodwithtech/dockle) で検査します。
dockle は CIS Docker Image Benchmark とコンテナイメージのベストプラクティスに基づく
イメージリンターで、root 実行・機密ファイルの混入・不適切なパーミッションなどを
チェックします。WARN 以上の指摘があると CI は失敗します(`--exit-code 1 --exit-level warn`)。

dockle が確実に読めるのは docker-archive 形式のため、image-builder が出力する
OCI アーカイブを skopeo で変換してから検査します。ローカルでは次のように実行できます。

```bash
sudo skopeo copy \
  oci-archive:output/almalinux-10.1-container-x86_64/almalinux-10.1-container-x86_64.tar \
  docker-archive:/tmp/dockle-input.tar:localhost/almalinux-custom:ci
sudo podman run --rm -v /tmp/dockle-input.tar:/input.tar:ro \
  goodwithtech/dockle:latest --exit-code 1 --exit-level warn \
  --ignore CIS-DI-0001 --input /input.tar
```

#### 除外している checkpoint

| Checkpoint | レベル | 除外理由 |
|---|---|---|
| [CIS-DI-0001](https://github.com/goodwithtech/dockle/blob/master/CHECKPOINT.md)(コンテナ用ユーザーの作成) | WARN | 本イメージは公式 `almalinux` イメージと同様のベース OS イメージであり、root が既定。blueprint の `container` タイプには OCI config の `USER` を設定する手段がなく、利用側の派生イメージで `USER` を設定する想定のため除外 |

なお INFO レベルとして CIS-DI-0005(Content Trust)、CIS-DI-0006(HEALTHCHECK)、
CIS-DI-0008(setuid/setgid ファイル)が報告されますが、INFO は失敗条件(WARN 以上)に
含まれないため除外設定は不要です。CIS-DI-0008 で列挙される setuid バイナリ(`su`,
`mount`, `passwd` など)は OS 標準のものです。

### コンテナレジストリ(GHCR)への push

スモークテスト通過後、イメージは `ghcr.io/ryo-aoki-pc/almalinux-container` へ
push されます。認証は組み込みの `GITHUB_TOKEN` を使用するため、追加のシークレット
設定は不要です。タグは次のルールで付与されます。

| トリガー | タグ |
|---|---|
| main への push | `latest` |
| 手動実行(workflow_dispatch) | ブランチ名(`/` は `-` に変換) |

push されたイメージは次のように利用できます。

```bash
podman pull ghcr.io/ryo-aoki-pc/almalinux-container:latest
podman run --rm ghcr.io/ryo-aoki-pc/almalinux-container:latest cat /etc/os-release
```

> **Note**: 初回 push 時に作成されるパッケージは既定で **private** です。公開する場合は
> GitHub のパッケージ設定(Package settings → Change visibility)で public に変更して
> ください。private のまま pull するには `podman login ghcr.io` で認証が必要です。

## 参照リンク

- [Image Builder ドキュメント](https://osbuild.org/docs/)
- [Blueprint Reference](https://osbuild.org/docs/user-guide/blueprint-reference/)
- [AlmaLinux 10.1 container イメージタイプ](https://osbuild.org/docs/user-guide/image-descriptions/almalinux-10.1/container/)
- [image-builder-cli](https://github.com/osbuild/image-builder-cli)
- [コマンドラインでのビルド(osbuild-composer)](https://osbuild.org/docs/on-premises/commandline/)
