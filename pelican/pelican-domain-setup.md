# Pelican Panel/Wings をドメインで公開して Minecraft に接続する手順（VPS 経由・ローカル側ポート開放不可）

この手順では、VPS を公開入口として使い、`frp`（Fast Reverse Proxy）でローカル VM（Wings ノード）へ安全にトンネルします。Cloudflare は DNS を管理します（HTTP 以外はプロキシ不可のため DNS のみ）。最終的に、`mc.cloudru.jp` などのドメインで Minecraft Java サーバに接続できるようにします。

- Panel は `pelican.cloudru.jp` で公開済み（Cloudflare）。
- VPS: 公開 IP 例 `158.51.109.155`。
- ローカル VM（Pelican/Wings）: `192.168.110.6`。パネルの Address 表示が `192.168.110.6:25568` などになっている。
- ローカル側はルータ/NATのため、外部からのポート開放は不可。

> 重要: Pelican の「Allocations」はノード上の“IP アドレスとポート”の割当です。ここに「ドメイン名」は表示されません（仕様）。ドメインは DNS で解決され、入口（VPS）へ到達後、`frp` 経由で Wings に転送されます。

---

## ゴール

- `mc.cloudru.jp` で Minecraft Java に接続可能（標準ポート 25565 を推奨）。
- Pelican の Allocations に `0.0.0.0:25565`（または使用中のポート）を追加し、該当サーバへ割り当て。

---

## クイックスタート（5分版・変数化）

以下の変数に自環境の値を設定して読み替えてください。

- `MC_DOMAIN`: 接続ドメイン（例: `mc.cloudru.jp`）
- `VPS_PUBLIC_IP`: VPS のグローバル IP（例: `158.51.109.155`）
- `FRP_TOKEN`: 強固な乱数トークン（非公開）
- `PRIMARY_PORT`: 公開ポート（例: `25570`）
- `ALLOW_RANGE`: 許可レンジ（例: `25565-26064` または `25500-25999`）
- `LOCAL_WINGS_IP`: ローカル Wings IP（例: `192.168.110.6`）
- `LOCAL_MC_PORT`: コンテナのローカル Minecraft ポート（例: `25565`/`25570`）

手順の流れ（要点のみ）：

1. Cloudflare DNS（DNS only）で `MC_DOMAIN → VPS_PUBLIC_IP` を設定。必要なら SRV でポートを隠す。
2. VPS に `frps` を導入し、`bind_port=7000` と `allow_ports=ALLOW_RANGE` を設定。
3. ローカル VM に `frpc` を導入し、`remote_port=PRIMARY_PORT` を `LOCAL_WINGS_IP:LOCAL_MC_PORT` へ転送。
4. Pelican のノードに `0.0.0.0:ALLOW_RANGE` を Allocation。対象サーバへ `PRIMARY_PORT` を割り当て Primary に設定。
5. 外部到達テスト `nc -vz MC_DOMAIN PRIMARY_PORT`、Minecraft クライアントで `MC_DOMAIN:PRIMARY_PORT` に接続。

---

## 構成概要

- Cloudflare DNS: `mc.cloudru.jp` を VPS の A レコードに向ける（Proxied: Off / DNS only）。
- VPS: `frps`（サーバ）を稼働。制御ポート `7000/tcp` でクライアントを受け、公開ポート `25565/tcp` を開ける。
- ローカル VM: `frpc`（クライアント）が VPS へ常時接続。VPSの `25565` をローカルの `192.168.110.6:<minecraft_port>` に転送。
- Pelican/Wings: 該当ポートをサーバへ割当（Allocations）。

---

## 1) Cloudflare DNS の設定

1. Cloudflare ダッシュボード → `cloudru.jp`。
2. A レコードを追加:
   - Name: `mc`
   - IPv4 address: `158.51.109.155`（VPS の公開 IP）
   - Proxy status: **DNS only（グレー雲）**
3. （任意）SRV レコードでポートを隠す場合:
   - Type: SRV
   - Service: `_minecraft`
   - Proto: `_tcp`
   - Name: `play`（例）
   - Target: `mc.cloudru.jp`
   - Port: `25565`（または 25568 を使うなら 25568）
   - Priority: `0`, Weight: `5`

> 注: Cloudflare のオレンジ雲（HTTP 逆プロキシ）は Minecraft の TCP を処理できません。非 HTTP は **DNS only** を使います。Cloudflare Spectrum を使う場合は別途契約が必要です。

---

## 2) VPS（`frps` サーバ）のセットアップ

以下は Debian/Ubuntu 例。別 OS はバイナリを置き換えてください。

### frps のインストール

```bash
sudo mkdir -p /opt/frp && cd /opt
sudo wget https://github.com/fatedier/frp/releases/download/v0.60.0/frp_0.60.0_linux_amd64.tar.gz
sudo tar xzf frp_0.60.0_linux_amd64.tar.gz
sudo mv frp_0.60.0_linux_amd64 frp
```

### 設定 `/etc/frp/frps.ini`

```ini
[common]
bind_port = 7000
authentication_method = token
token = ${FRP_TOKEN}
; 公開可能ポートを絞る（推奨）
allow_ports = ${ALLOW_RANGE}
; 例: 25565,25570 / 25500-25999 / 25565-26064
; 注: allow_ports は "許可範囲" の指定であり、frpc が該当 remote_port を作成・接続したときだけ
; frps 側に待受が生成されます。範囲指定だけでは自動公開されません。
```

### systemd ユニット `/etc/systemd/system/frps.service`

```ini
[Unit]
Description=frp server (frps)
After=network.target

[Service]
Type=simple
ExecStart=/opt/frp/frps -c /etc/frp/frps.ini
Restart=always

[Install]
WantedBy=multi-user.target
```

### 起動とファイアウォール

```bash
sudo ufw allow 7000/tcp
# 500ポート範囲をまとめて許可（Ubuntu/UFW）
sudo ufw allow 25500:25999/tcp
# 25565から500ポートの場合はこちら
# sudo ufw allow 25565:26064/tcp

sudo systemctl daemon-reload
sudo systemctl enable --now frps
```

---

## 3) ローカル VM（`frpc` クライアント）のセットアップ

### frpc のインストール

```bash
sudo mkdir -p /opt/frp && cd /opt
sudo wget https://github.com/fatedier/frp/releases/download/v0.60.0/frp_0.60.0_linux_amd64.tar.gz
sudo tar xzf frp_0.60.0_linux_amd64.tar.gz
sudo mv frp_0.60.0_linux_amd64 frp
```

### 設定 `/etc/frp/frpc.ini`

Minecraft の実ポートに合わせて `LOCAL_MC_PORT` を設定します。

```ini
[common]
server_addr = ${VPS_PUBLIC_IP}    ; または MC_DOMAIN（DNS only）
server_port = 7000
authentication_method = token
token = ${FRP_TOKEN}

[minecraft-${PRIMARY_PORT}]
type = tcp
local_ip = ${LOCAL_WINGS_IP}
local_port = ${LOCAL_MC_PORT}
remote_port = ${PRIMARY_PORT}
```

> 注意: `frpc` は「1セクション = 1公開ポート」です。単一の `[minecraft]` セクションでは範囲一括割当はできません。複数サーバを同時公開する場合は、ポートごとにセクションを追加するか、末尾の「10) 追加: frpc を使って 25500-25999 を一括公開する（自動生成）」のスクリプトで自動生成してください。

### systemd ユニット `/etc/systemd/system/frpc.service`

```ini
[Unit]
Description=frp client (frpc)
After=network.target

[Service]
Type=simple
ExecStart=/opt/frp/frpc -c /etc/frp/frpc.ini
Restart=always

[Install]
WantedBy=multi-user.target
```

### 起動

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frpc
```

> ログ確認: `journalctl -u frpc -f`（接続成功/失敗を確認）。

---

## 4) Pelican（Wings）での Allocations 設定

1. パネル左メニュー → Admin → Nodes → ローカルノード選択 → Allocations。
2. 「Create Allocation」で以下を追加:
   - IP Address: `0.0.0.0`（全インターフェイスで待受）
   - Ports: `25500-25999`（例。まとめて500ポートを割当可能）
     または `25565-26064`（25565から500ポート開始の例）
3. 対象サーバ → Network → 追加した Allocation を割り当て、Primary に設定。

> 注: ここに「mc.cloudru.jp」は表示されません。Allocations は **Wings ノード上の IP/ポート** に対する割当であり、ドメインは DNS レベルで入口（VPS）に向ける設計です。

---

## 5) 動作確認

- まず VPS で待受を確認:

```bash
sudo ss -lntp | grep 25565
```

- ローカル VMで `frpc` が接続済みかログ確認:

```bash
journalctl -u frpc -f
```

- 外部クライアントから接続テスト（Linux/macOS例）:

```bash
nc -vz mc.cloudru.jp 25565
```

- Minecraft クライアントで `mc.cloudru.jp` に接続（SRV を設定した場合は `play.cloudru.jp` のみでも可）。

---

## 6) よくある躓きポイントと対処

- Cloudflare の「オレンジ雲」を有効にすると非 HTTP トラフィックが遮断されます。**DNS only** にしてください。
- `frps` の `allow_ports` を設定しているのに別ポートを指定すると失敗します。公開するポートだけを列挙。
- Wing 側ポートがコンテナに割り当てられていないと接続できません。サーバの Network タブで割当済みか確認。
- Bedrock Edition（UDP）を使う場合は、`type = udp` のエントリを追加し、`19132/udp` などを同様に転送します（Cloudflare は UDP もプロキシ不可のため DNS only）。

---

## 7) 任意: ポート番号を隠したい（SRV レコード）

- `mc.cloudru.jp` で 25568 を使いつつ、`play.cloudru.jp` でポート無し接続を可能にする例:
  - A: `mc.cloudru.jp` → VPS `158.51.109.155`（DNS only）
  - SRV: `_minecraft._tcp.play.cloudru.jp` → Target: `mc.cloudru.jp`, Port: `25568`
  - クライアントは `play.cloudru.jp` のみで接続可能。

---

## 8) 仕様上の補足（Allocations とドメイン）

- Pelican/Wings の Allocations は **ノードのローカルインターフェイス**（例: `192.168.110.6`, `172.17.0.1`, `0.0.0.0`）のみを表示します。DNS 名は選択できません。
- ドメインを使いたい場合は、DNS → VPS（公開入口）→ `frp` でローカルへ転送、というレイヤ分離で実現します。
- もし VPS 側で Wings を動かす場合は、VPS ノードの Allocations として公開 IP を直接選べます（別ノード構成）。

---

## 9) まとめ

- Cloudflare DNS で `mc.cloudru.jp` を VPS に向ける（DNS only）。
- VPS に `frps`、ローカル VM に `frpc` を設定し、`25565`（または任意ポート）をトンネル。
- Pelican で `0.0.0.0:<port>` をサーバに割り当て。
- クライアントはドメインで接続可能。Allocations にドメインは表示されないのが正常。

---

## 10) 追加: frpc を使って 25500-25999 を一括公開する（自動生成）

大量のポートをまとめて公開したい場合は、`frpc.ini` のプロキシセクションを自動生成します。以下は 25500-25999 の 500 ポートをすべて同じローカルサーバ（例: `192.168.110.6:25565`）へ転送する例です。

### 10.1 単一ローカルサーバへ 500 ポートを一括転送

```bash
TOKEN="<同じトークン>"
ADDR="158.51.109.155"
LOCAL_IP="192.168.110.6"
LOCAL_PORT=25565       # 必要なら 25568 などに変更
START=25500
END=25999

sudo tee /etc/frp/frpc.ini > /dev/null <<EOF
[common]
server_addr = ${ADDR}
server_port = 7000
authentication_method = token
token = ${TOKEN}
EOF

for p in $(seq ${START} ${END}); do
   sudo tee -a /etc/frp/frpc.ini > /dev/null <<EOF
[minecraft-${p}]
type = tcp
local_ip = ${LOCAL_IP}
local_port = ${LOCAL_PORT}
remote_port = ${p}
EOF
done

sudo systemctl restart frpc
```

> 注: この方法は「同じローカルサーバへ複数ポートで到達可能」にする用途です。複数サーバを公開したい場合は次の CSV 方式を使います。

### 10.2 複数サーバ（IP/ポート）を CSV で一括生成

1. マッピングファイルを作成（例 `/etc/frp/mappings.csv`）。各行は `remote_port,local_ip,local_port`。
   ```csv
   25501,192.168.110.6,25565
   25502,192.168.110.6,25568
   25575,192.168.110.6,25575
   ```
2. 生成スクリプトを実行:

   ```bash
   TOKEN="<同じトークン>"
   ADDR="158.51.109.155"
   MAP="/etc/frp/mappings.csv"

   sudo tee /etc/frp/frpc.ini > /dev/null <<EOF
   [common]
   server_addr = ${ADDR}
   server_port = 7000
   authentication_method = token
   token = ${TOKEN}
   EOF

   while IFS="," read -r remote ip port; do
      [ -z "${remote}${ip}${port}" ] && continue
      sudo tee -a /etc/frp/frpc.ini > /dev/null <<EOF
   [minecraft-${remote}]
   type = tcp
   local_ip = ${ip}
   local_port = ${port}
   remote_port = ${remote}
   EOF
   done < "${MAP}"

   sudo systemctl restart frpc
   ```

### 10.3 Pelican 側の割当

- ノードの Allocations で `IP: 0.0.0.0`, `Ports: 25500-25999` を作成済みであれば、各サーバに必要なポート（CSVの `remote_port`）を割り当ててください。

### 10.4 VPS ファイアウォール

```bash
sudo ufw allow 25500:25999/tcp
# Bedrock（UDP）も扱うなら
sudo ufw allow 25500:25999/udp
```

### 10.5 運用上の注意

- `frps` の `allow_ports = 25500-25999` に一致しないポートは公開されません。
- ポート重複（同じ `remote_port` の二重定義）は失敗します。CSV で一意性を担保してください。
- 生成後は `journalctl -u frpc -f` で接続状況を確認し、`ss -lntp`（TCP）/`ss -lnup`（UDP）で待受を確認します。

---

## 11) 運用チェックリスト（再現用）

- Cloudflare: A レコードが DNS only、SRV の Target/Port が正しい
- frps: `allow_ports` に公開したいポートが含まれる、`systemctl is-active frps` が active
- frpc: `token` 一致、`remote_port` が意図した値、`systemctl is-active frpc` が active
- Firewall: VPS で `PRIMARY_PORT/tcp` が許可されている
- Pelican: ノードに `0.0.0.0` の割当があり、対象サーバに `PRIMARY_PORT` が割当＆Primary
- コンテナ: サーバが Running、Address が `0.0.0.0:PRIMARY_PORT`（例: `0.0.0.0:25570`）

---

## 12) 検証例（成功パターン）

- サーバ `mc02` を `PRIMARY_PORT=25570` で起動（Address: `0.0.0.0:25570`）。
- 外部到達テスト: `nc -vz MC_DOMAIN 25570` → `succeeded`。
- Minecraft クライアント: `MC_DOMAIN:25570` で入室可能。
