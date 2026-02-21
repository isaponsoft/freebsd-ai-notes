# FreeBSD 15における階層化Jail（Podman x Native）構築メモ

## 1. 概要

FreeBSDホスト（Tamamo）上でPodmanを動かし、そのコンテナ（1層目）内部でさらにFreeBSDネイティブのJail（2層目）を起動する「入れ子」構造の構築手順。

## 2. ホスト側での準備

巨大なホストOSデータを吸い出さないよう、公式の最小ベースシステムからPodmanイメージを作成する。

```bash
# 最小ベースシステムの取得とインポート
pkg install podman
fetch https://download.freebsd.org/releases/amd64/15.0-RELEASE/base.txz
cat base.txz | sudo podman import - freebsd:15-minimal

```

## 3. 1層目（Podmanコンテナ）の起動

特権モードで起動し、ネットワークを引き継ぐ。

```bash
sudo podman run -it --rm \
  --name nested-boss \
  --privileged \
  -v /etc/resolv.conf:/etc/resolv.conf:ro \
  freebsd:15-minimal /bin/sh

```

## 4. ホスト側からの「入れ子許可」の注入

コンテナが起動した直後、ホスト（Tamamo）の別ターミナルからJIDを指定して、子Jailの作成権限を付与する。

```bash
# JIDを確認
jls

# 子Jailの作成とマウントを許可（JID 20の場合）
sudo sysctl security.jail.allow_raw_sockets=1
sudo sysctl security.jail.children_max=10
sudo jail -m jid=20 \
  children.max=10 \
  allow.mount=1 \
  allow.mount.nullfs=1 \
```

## 5. 2層目（Native Jail）の構築

1層目のコンテナ内部で、自身のファイルシステムを再利用してJailを生成する。

```bash
# 1層目コンテナ内での操作
# 2層目（inner-world）の起動
jail -c name=recursive_jail path=/ host.hostname=inner-world command=/bin/sh

```

## 6. 検証と隔離の確認

2層目に入った後、プロセス空間が隔離されていることを確認する。

```bash
# 2層目内での実行結果例
# ps ax
#  PID TT  STAT    TIME COMMAND
# 28271  7  SJ     0:00.01 /bin/sh
# 28272  7  R+J    0:00.00 ps ax

```

---

### 技術的ポイント

* **children.max**: これを設定しない限り、Jail内での `jail -c` は `Operation not permitted` となる。
* **path=/**: 1層目のルートをそのまま使うことで、最小限のバイナリコピーで2層目を実現。
* **STAT列の 'J'**: プロセスがJail（またはその子Jail）に閉じ込められている証。
