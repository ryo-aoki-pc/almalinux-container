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

ビルドが完了すると `output/almalinux-10.1-container-x86_64/container.tar`
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
sudo skopeo copy oci-archive:output/almalinux-10.1-container-x86_64/container.tar \
  containers-storage:localhost/almalinux-custom:latest

# 実行確認
sudo podman run --rm localhost/almalinux-custom:latest cat /etc/os-release
sudo podman run --rm -it localhost/almalinux-custom:latest tmux
```

`podman load -i container.tar` でも取り込み可能です(タグは付与されません)。

## CI(GitHub Actions)

`.github/workflows/build-container.yml` が main への push、pull request、手動実行
(workflow_dispatch)をトリガーに以下を行います。

1. image-builder-cli コンテナで `blueprint.toml` から AlmaLinux 10.1 の
   コンテナイメージをビルド
2. スモークテスト: イメージを取り込み、`/etc/os-release` が AlmaLinux であることを確認
3. `container.tar` をワークフローアーティファクトとしてアップロード

### 発展: レジストリへの公開

ビルドしたイメージを GHCR(GitHub Container Registry)へ公開する場合は、ワークフローに
`permissions: packages: write` を追加し、スモークテスト後に次のようなステップを追加します。

```bash
sudo skopeo copy \
  --dest-creds "${{ github.actor }}:${{ secrets.GITHUB_TOKEN }}" \
  "oci-archive:<container.tar のパス>" \
  "docker://ghcr.io/<owner>/almalinux-custom:latest"
```

## 参照リンク

- [Image Builder ドキュメント](https://osbuild.org/docs/)
- [Blueprint Reference](https://osbuild.org/docs/user-guide/blueprint-reference/)
- [AlmaLinux 10.1 container イメージタイプ](https://osbuild.org/docs/user-guide/image-descriptions/almalinux-10.1/container/)
- [image-builder-cli](https://github.com/osbuild/image-builder-cli)
- [コマンドラインでのビルド(osbuild-composer)](https://osbuild.org/docs/on-premises/commandline/)
