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
| `blueprint.toml` | 通常(default)イメージの定義(パッケージ・カスタマイズ) |
| `blueprint-minimal.toml` | minimal イメージの定義(追加パッケージなし) |
| `blueprint-init.toml` | init イメージの定義(systemd + コンテナ向けユニットマスク) |
| `blueprint-micro.toml` | micro イメージの定義(パッケージマネージャなし) |
| `.github/workflows/build-container.yml` | GitHub Actions による自動ビルド(default / minimal / init / micro を matrix で並行ビルド) |

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
  build container --distro almalinux-10.2 --blueprint /blueprint.toml
```

ビルドが完了すると `output/almalinux-10.2-container-x86_64/almalinux-10.2-container-x86_64.tar`
(OCI アーカイブ)が生成されます。

利用可能なディストリビューションとイメージタイプの一覧は次のコマンドで確認できます。

```bash
sudo podman run --rm ghcr.io/osbuild/image-builder-cli:latest list
```

### minimal イメージ(almalinux-minimal 相当)

`container-minimal` イメージタイプを使うと、公式の `almalinux/10-minimal` に相当する
microdnf ベースの最小イメージをビルドできます。ベースパッケージセットは
`bash` / `coreutils-single` / `redhat-release` / `rpm` / `microdnf` のみで、
dnf 本体や Python を含みません(`grep` などの一般コマンドも入りません)。

```bash
sudo podman run --rm --privileged \
  -v "$PWD/output:/output" \
  -v "$PWD/blueprint-minimal.toml:/blueprint.toml:ro" \
  ghcr.io/osbuild/image-builder-cli:latest \
  build container-minimal --distro almalinux-10.2 --blueprint /blueprint.toml
```

出力は `output/almalinux-10.2-container-minimal-x86_64/almalinux-10.2-container-minimal-x86_64.tar`
です。コンテナ内でのパッケージ追加は `microdnf install <pkg>` で行います。

### init イメージ(almalinux-init 相当)

systemd を PID 1 として起動し、コンテナ内で systemd サービスを動かすための
イメージです。公式の `almalinux/10-init` に相当します。

- `blueprint-init.toml` で `systemd` / `procps-ng` を追加し、コンテナ内で動作しない
  ユニット(`systemd-udevd.service`、`getty.target` など10個)を公式イメージと同じ
  構成で `[customizations.services]` の `masked` によりマスクしています
- blueprint では OCI config(`CMD` / `STOPSIGNAL`)を設定できないため、CI では
  ビルド後に buildah で `CMD /sbin/init` と `STOPSIGNAL SIGRTMIN+3` を書き込んでいます
  (`.github/workflows/build-container.yml` の「Configure init image OCI metadata」
  ステップを参照)
- CI は実際にコンテナを起動し、`systemctl is-system-running` で systemd が
  起動することを検証します

```bash
# systemd サービスを動かすコンテナとして起動
podman run -d --name svc ghcr.io/ryo-aoki-pc/almalinux-container:init
podman exec svc systemctl status
podman stop svc   # SIGRTMIN+3 で systemd が正常シャットダウン
```

### micro イメージ(almalinux-micro 相当)

パッケージマネージャ(dnf / microdnf / rpm)を一切含まない最小ランタイムイメージで、
公式の `almalinux/10-micro` に相当します。ビルド済みバイナリを載せるベースイメージ
としての利用を想定しています。

- osbuild には micro 用のイメージタイプが存在しないため、CI の
  「Prepare distro definitions」ステップで `container-micro` イメージタイプを
  distro 定義に追記して定義しています(ベースセット:
  `bash` / `coreutils-single` / `glibc-minimal-langpack` / `redhat-release`)
- blueprint(`blueprint-micro.toml`)は追加パッケージなし。パッケージマネージャが
  ないため `dnf.conf` も配置しません
- 公式 micro は RPM データベースも削除していますが、本イメージでは脆弱性スキャナが
  パッケージ情報を読めるよう rpmdb を残しています(rpm コマンド自体は含まれません)

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
sudo skopeo copy oci-archive:output/almalinux-10.2-container-x86_64/almalinux-10.2-container-x86_64.tar \
  containers-storage:localhost/almalinux-custom:latest

# 実行確認
sudo podman run --rm localhost/almalinux-custom:latest cat /etc/os-release
sudo podman run --rm -it localhost/almalinux-custom:latest tmux
```

`podman load -i <ビルドした .tar>` でも取り込み可能です(タグは付与されません)。

## 推奨パッケージ(weak dependencies)の無効化

このリポジトリのイメージは、dnf の推奨パッケージ(weak dependencies / Recommends)を
インストールしません。3つの層で制御しています。

| 層 | 仕組み |
|---|---|
| blueprint で追加するパッケージ | images ライブラリの仕様により、もともと weak deps 無効で依存解決される(設定不要) |
| イメージタイプ固有のベースパッケージセット | distro 既定は `install_weak_deps: true`。blueprint では上書きできないため、CI で distro 定義を取得・パッチし、実験的な `IMAGE_BUILDER_EXPERIMENTAL=yamldir=` 上書きで `false` を適用 |
| イメージ内での将来の `dnf install`(派生イメージ含む) | blueprint の `[[customizations.files]]` で `/etc/dnf/dnf.conf` に `install_weak_deps=False` を設定 |

ベースセット用の distro 定義パッチは、`image-builder version` が報告する images
ライブラリのバージョンと**同一リビジョン**の定義を取得して適用するため、
定義スキーマの不整合は発生しません(手順は `.github/workflows/build-container.yml` の
「Prepare distro definitions with weak deps disabled」ステップを参照)。
ローカルでビルドする場合も同じ手順で `/tmp/distrodefs` を用意し、次のように
オプションを追加してください。

```bash
sudo podman run --rm --privileged \
  -v "$PWD/output:/output" \
  -v "$PWD/blueprint.toml:/blueprint.toml:ro" \
  -v /tmp/distrodefs:/distrodefs:ro \
  -e IMAGE_BUILDER_EXPERIMENTAL=yamldir=/distrodefs \
  ghcr.io/osbuild/image-builder-cli:latest \
  build container --distro almalinux-10.2 --blueprint /blueprint.toml
```

> **Note**: `yamldir` は images ライブラリの実験的機能です。将来の image-builder で
> 動作が変わる可能性がありますが、CI ではバージョン整合を取って取得しているため、
> 壊れた場合もビルド失敗として即座に検知できます。

## CI(GitHub Actions)

`.github/workflows/build-container.yml` が main への push、pull request、手動実行
(workflow_dispatch)をトリガーに、4つのバリアントを matrix で並行ビルドします。

| バリアント | イメージタイプ | blueprint |
|---|---|---|
| default | `container` | `blueprint.toml` |
| minimal | `container-minimal` | `blueprint-minimal.toml` |
| init | `container` + OCI 設定書き換え | `blueprint-init.toml` |
| micro | `container-micro`(CI で定義を追加) | `blueprint-micro.toml` |

各ジョブは以下を行います。

1. distro 定義を取得し、推奨パッケージ(weak deps)を無効化するパッチを適用
2. image-builder-cli コンテナで対応する blueprint から AlmaLinux 10.2 の
   コンテナイメージをビルド
3. 正規化: コンテナに必要なファイル/ディレクトリを作成し、不要なものを削除して
   単一レイヤーに squash(init は systemd 用の OCI 設定もここで付与)
4. スモークテスト: `/etc/os-release` が AlmaLinux であること、`dnf.conf` の
   `install_weak_deps=False`、作成したファイルの存在・不要ファイルの削除を確認し、
   インストール済みパッケージ数を表示
5. lint: dockle でイメージがベストプラクティスに従っているか検査(WARN 以上で失敗)
6. ビルドした `.tar`(OCI アーカイブ)をワークフローアーティファクトとしてアップロード
7. GHCR(GitHub Container Registry)へ push(pull request 時は実行しない)

### イメージの正規化(ファイルの作成・削除)

ビルド直後のイメージにはパッケージマネージャのキャッシュやビルド時のログなど
コンテナに不要なものが残り、逆にコンテナ実行時に必要な空ファイルが用意されて
いないことがあります。ワークフローの「Normalize image」ステップで buildah により
rootfs を調整し、`buildah commit --squash` で単一レイヤーに固めて削除を実サイズに
反映します(公式 AlmaLinux コンテナイメージと同様の処理)。

**作成するファイル/ディレクトリ:**

| パス | 目的 |
|---|---|
| `/etc/machine-id`(空) | 初回起動時に systemd が一意の ID を生成できるようにする |
| `/etc/resolv.conf`(空) | コンテナランタイムが起動時に内容を流し込む |
| `/etc/hostname`(空) | 同上 |
| `/tmp`, `/var/tmp`(1777) | 書き込み可能な一時ディレクトリ |
| `/run`(0755) | 実行時の状態ディレクトリ |

**削除するファイル/ディレクトリ:**

| パス | 理由 |
|---|---|
| `/var/cache/dnf`, `/var/cache/yum`, `/var/cache/ldconfig` | 再生成可能なキャッシュ |
| `/var/log/` 配下のファイル | ビルド時のログ |
| `/tmp`, `/var/tmp` の中身 | ビルドで残った一時ファイル |
| `/boot` の中身 | コンテナではカーネル/ブート関連は不要 |

RPM データベース(`/var/lib/rpm`)と dnf 履歴(`/var/lib/dnf`)は、パッケージ
マネージャと脆弱性スキャナが動作し続けられるよう残しています(micro はもともと
これらを含みません)。

### イメージの lint(dockle)

ビルドしたイメージは [dockle](https://github.com/goodwithtech/dockle) で検査します。
dockle は CIS Docker Image Benchmark とコンテナイメージのベストプラクティスに基づく
イメージリンターで、root 実行・機密ファイルの混入・不適切なパーミッションなどを
チェックします。WARN 以上の指摘があると CI は失敗します(`--exit-code 1 --exit-level warn`)。

dockle が確実に読めるのは docker-archive 形式のため、image-builder が出力する
OCI アーカイブを skopeo で変換してから検査します。ローカルでは次のように実行できます。

```bash
sudo skopeo copy \
  oci-archive:output/almalinux-10.2-container-x86_64/almalinux-10.2-container-x86_64.tar \
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

| トリガー | default | minimal | init | micro |
|---|---|---|---|---|
| main への push | `latest` | `minimal` | `init` | `micro` |
| 手動実行(workflow_dispatch) | ブランチ名(`/` は `-` に変換) | ブランチ名 + `-minimal` | ブランチ名 + `-init` | ブランチ名 + `-micro` |

push されたイメージは次のように利用できます。

```bash
podman pull ghcr.io/ryo-aoki-pc/almalinux-container:latest
podman run --rm ghcr.io/ryo-aoki-pc/almalinux-container:latest cat /etc/os-release

# minimal バリアント
podman pull ghcr.io/ryo-aoki-pc/almalinux-container:minimal
podman run --rm ghcr.io/ryo-aoki-pc/almalinux-container:minimal microdnf --version
```

> **Note**: 初回 push 時に作成されるパッケージは既定で **private** です。公開する場合は
> GitHub のパッケージ設定(Package settings → Change visibility)で public に変更して
> ください。private のまま pull するには `podman login ghcr.io` で認証が必要です。

## 参照リンク

- [Image Builder ドキュメント](https://osbuild.org/docs/)
- [Blueprint Reference](https://osbuild.org/docs/user-guide/blueprint-reference/)
- [Image Builder がサポートするイメージタイプ一覧](https://osbuild.org/docs/user-guide/image-descriptions/)
- [image-builder-cli](https://github.com/osbuild/image-builder-cli)
- [コマンドラインでのビルド(osbuild-composer)](https://osbuild.org/docs/on-premises/commandline/)
