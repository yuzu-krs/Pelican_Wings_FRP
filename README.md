# Pelican/Wings + FRP で Minecraft をドメイン公開する完全手順（NAT越え・トークン非公開）

この手順書は、ローカルの Wings ノード（ポート解放不可）を、VPS 経由の FRP トンネルで外部公開し、Cloudflare の DNS でドメイン接続できるようにするものです。実運用で検証済みの流れを、機微情報を含めない形で再構成しています。

---

## ゴール

- `MC_DOMAIN`（例: `mc.example.com`）で Minecraft Java 版に接続できること。
- Wings のサーバに `PRIMARY_PORT`（例: `25570`）を割り当て、Primary として利用。
- `frps`/`frpc` は systemd で常時稼働し、必要ポートのみ公開。

---

## 前提

- ドメイン管理: Cloudflare（HTTP 以外は DNS only）
- 公開入口: VPS（グローバル IP 所有）
- サーバ稼働: ローカル VM 上の Pelican/Wings（NAT 配下）
- FRP バージョン例: v0.60.0
- 変数（必要に応じて読み替え）
  - `MC_DOMAIN`: Minecraft 接続に使うドメイン（例: `mc.example.com`）
  - `VPS_PUBLIC_IP`: VPS のグローバル IP
  - `FRP_TOKEN`: 強固な共有トークン（非公開・乱数）
  - `PRIMARY_PORT`: 外部公開ポート（例: `25570`）
  - `ALLOW_RANGE`: 許可レンジ（例: `25500-25999` または `25565-26064`）
  - `LOCAL_WINGS_IP`: ローカル Wings の IP（例: `192.168.110.6`）
  - `LOCAL_MC_PORT`: コンテナが使うローカル Minecraft ポート（例: `25565`/`25570`/任意）

---

## クイックスタート（5分版）

1. 変数を決める（例）

```bash
export MC_DOMAIN="mc.example.com"
export VPS_PUBLIC_IP="203.0.113.10"
export FRP_TOKEN="<強固な乱数>"   # 公開しない
export PRIMARY_PORT=25570
export ALLOW_RANGE="25565-26064"  # 単一でも可: "25570"
export LOCAL_WINGS_IP="192.168.110.6"
export LOCAL_MC_PORT=25570         # コンテナの実ポートに合わせる
```

2. Cloudflare DNS（ダッシュボード）


    - A: `mc` → `VPS_PUBLIC_IP`、Proxy: DNS only
    - （任意）SRV: `_minecraft._tcp.play` → Target: `MC_DOMAIN`, Port: `PRIMARY_PORT`

3. VPS に frps を最小構成で導入

```bash
sudo mkdir -p /opt/frp && cd /opt
sudo wget https://github.com/fatedier/frp/releases/download/v0.60.0/frp_0.60.0_linux_amd64.tar.gz
sudo tar xzf frp_0.60.0_linux_amd64.tar.gz && sudo mv frp_0.60.0_linux_amd64 frp

sudo mkdir -p /etc/frp
sudo tee /etc/frp/frps.ini > /dev/null <<EOF
[common]
bind_port = 7000
authentication_method = token
token = ${FRP_TOKEN}
allow_ports = ${ALLOW_RANGE}
EOF

sudo tee /etc/systemd/system/frps.service > /dev/null <<'EOF'
[Unit]
Description=frp server (frps)
After=network.target
[Service]
Type=simple
ExecStart=/opt/frp/frps -c /etc/frp/frps.ini
Restart=always
[Install]
WantedBy=multi-user.target
EOF

sudo ufw allow 7000/tcp && sudo ufw allow ${PRIMARY_PORT}/tcp || true
sudo systemctl daemon-reload && sudo systemctl enable --now frps
```

4. ローカル VM に frpc を導入（単一公開）

```bash
sudo mkdir -p /opt/frp && cd /opt
sudo wget https://github.com/fatedier/frp/releases/download/v0.60.0/frp_0.60.0_linux_amd64.tar.gz
sudo tar xzf frp_0.60.0_linux_amd64.tar.gz && sudo mv frp_0.60.0_linux_amd64 frp

sudo mkdir -p /etc/frp
sudo tee /etc/frp/frpc.ini > /dev/null <<EOF
[common]
server_addr = ${VPS_PUBLIC_IP}
server_port = 7000
authentication_method = token
token = ${FRP_TOKEN}

[minecraft-${PRIMARY_PORT}]
type = tcp
local_ip = ${LOCAL_WINGS_IP}
local_port = ${LOCAL_MC_PORT}
remote_port = ${PRIMARY_PORT}
EOF

sudo tee /etc/systemd/system/frpc.service > /dev/null <<'EOF'
[Unit]
Description=frp client (frpc)
After=network.target
[Service]
Type=simple
ExecStart=/opt/frp/frpc -c /etc/frp/frpc.ini
Restart=always
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload && sudo systemctl enable --now frpc
```

5. Pelican/Wings


    - ノード Allocations: `0.0.0.0` + `ALLOW_RANGE`（または `PRIMARY_PORT`）
    - 対象サーバに `PRIMARY_PORT` を割当 → Primary に設定

6. 検証

```bash
nc -vz ${MC_DOMAIN} ${PRIMARY_PORT}
```

Minecraft クライアント: `MC_DOMAIN:${PRIMARY_PORT}`（SRVなら `play.example.com`）

---

## 構成概要

1. Cloudflare DNS: `MC_DOMAIN` を VPS の A レコードに向ける（DNS only）。
2. VPS: `frps`（サーバ）を起動。`7000/tcp` で制御、`PRIMARY_PORT/tcp` を公開。
3. ローカル VM: `frpc`（クライアント）が VPS に常時接続し、`PRIMARY_PORT` → `LOCAL_WINGS_IP:LOCAL_MC_PORT` を転送。
4. Pelican/Wings: ノードに `0.0.0.0:ALLOW_RANGE` を Allocation し、対象サーバへ `PRIMARY_PORT` を割当（Primary）。

---

## 1) Cloudflare DNS 設定

1. ゾーン選択 → DNS → レコード追加。
2. A レコード:
   - Name: `mc`（例）
   - IPv4 address: `VPS_PUBLIC_IP`
   - Proxy status: DNS only（グレー雲）
3. （任意）SRV レコードでポートを隠す:
   - Type: SRV
   - Service: `_minecraft`
   - Proto: `_tcp`
   - Name: `play`（例）
   - Target: `MC_DOMAIN`
   - Port: `PRIMARY_PORT`

注意: Cloudflare の HTTP プロキシ（オレンジ雲）は Minecraft の TCP を扱えません。Spectrum を使う場合は別契約が必要です。

---

## 2) VPS（frps サーバ）のセットアップ

インストール（Debian/Ubuntu 例）:

```bash
sudo mkdir -p /opt/frp && cd /opt
sudo wget https://github.com/fatedier/frp/releases/download/v0.60.0/frp_0.60.0_linux_amd64.tar.gz
sudo tar xzf frp_0.60.0_linux_amd64.tar.gz
sudo mv frp_0.60.0_linux_amd64 frp
```

設定 `/etc/frp/frps.ini`:

```ini
[common]
bind_port = 7000
authentication_method = token
token = ${FRP_TOKEN}
allow_ports = ${ALLOW_RANGE}
; 例: allow_ports = 25565,25570 または 25500-25999 / 25565-26064
```

systemd ユニット `/etc/systemd/system/frps.service`:

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

ファイアウォール（UFW 例）:

```bash
sudo ufw allow 7000/tcp
sudo ufw allow 25500:25999/tcp    # レンジ利用時の例
# または sudo ufw allow 25565:26064/tcp
```

起動:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frps
```

---

## 3) ローカル VM（frpc クライアント）のセットアップ

インストール（Debian/Ubuntu 例）:

```bash
sudo mkdir -p /opt/frp && cd /opt
sudo wget https://github.com/fatedier/frp/releases/download/v0.60.0/frp_0.60.0_linux_amd64.tar.gz
sudo tar xzf frp_0.60.0_linux_amd64.tar.gz
sudo mv frp_0.60.0_linux_amd64 frp
```

設定 `/etc/frp/frpc.ini`（単一ポート公開の基本形）:

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

systemd ユニット `/etc/systemd/system/frpc.service`:

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

起動/ログ:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frpc
journalctl -u frpc -f
```

メモ: `frpc` は「1セクション=1公開ポート」。複数サーバや多数ポートは次節の自動生成を利用。

---

## 4) Pelican/Wings の Allocations とサーバ設定

1. Admin → Nodes → 対象ノード → Allocations → Create。
2. IP Address: `0.0.0.0`
3. Ports: `ALLOW_RANGE`（例: `25500-25999` または `25565-26064`）
4. 対象サーバ → Network → 必要ポートを割当 → Primary に設定（例: `PRIMARY_PORT = 25570`）。

注意: Allocations はノードの IP/ポートの割当。ドメインは DNS で入口（VPS）へ向ける設計です。

---

## 5) 動作確認（外部到達とクライアント接続）

VPS で待受確認:

```bash
sudo ss -lntp | grep ${PRIMARY_PORT}
```

ローカル VM で frpc 接続確認:

```bash
journalctl -u frpc -f
```

外部到達テスト:

```bash
nc -vz ${MC_DOMAIN} ${PRIMARY_PORT}
```

Minecraft クライアント:

- サーバーアドレス: `MC_DOMAIN:${PRIMARY_PORT}`（SRV 設定時は `play.example.com` などポート省略可）

---

## 6) 大量ポート公開（自動生成）

単一ローカルサーバへレンジ一括転送（例: `25500-25999` を同じ `LOCAL_MC_PORT` へ）:

```bash
TOKEN="${FRP_TOKEN}"
ADDR="${VPS_PUBLIC_IP}"
LOCAL_IP="${LOCAL_WINGS_IP}"
LOCAL_PORT=${LOCAL_MC_PORT}
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

複数サーバを CSV で一括生成（`remote_port,local_ip,local_port`）:

```csv
25501,192.168.110.6,25565
25502,192.168.110.6,25568
25575,192.168.110.6,25575
```

生成スクリプト例:

```bash
TOKEN="${FRP_TOKEN}"
ADDR="${VPS_PUBLIC_IP}"
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

Pelican 側では、ノードに `0.0.0.0:ALLOW_RANGE` が割当済みであることを確認し、各サーバに必要な `remote_port` を割当ててください。

---

## 7) トラブルシューティング

- Cloudflare がオレンジ雲だと接続不可。DNS only にする。
- `frps` の `allow_ports` にない `remote_port` を使うと失敗する。
- Pelican でポートが別サーバに割当済みだとコンテナが起動できない（Port already in use）。割当の整理か別ポートへ変更。
- `frpc` はセクション単位。レンジ指定しても自動で多数の待受は作られない。必要分を生成する。
- Bedrock（UDP）は `type = udp` と `19132/udp` 等を別途許可。Cloudflareは UDP もプロキシ不可。

---

## 8) セキュリティ/運用の要点

- `FRP_TOKEN` は強固な値を使い、構成ファイルの権限を絞る。
- 公開ポートは必要最小限（`allow_ports` で絞る）。
- `Restart=always` で自動復帰。`journalctl -u frps|frpc -f` で監視。
- 変更時は `sudo systemctl daemon-reload` を忘れずに。

---

## 8.1 運用チェックリスト（再現用）

- Cloudflare: A レコードが DNS only、SRVの Target/Port が正しい
- frps: `allow_ports` に公開したいポートが含まれる、`systemctl is-active frps` が active
- frpc: `token` 一致、`remote_port` が意図した値、`systemctl is-active frpc` が active
- Firewall: VPS で `PRIMARY_PORT/tcp` が許可されている
- Pelican: ノードに `0.0.0.0` の割当があり、対象サーバに `PRIMARY_PORT` が割当＆Primary
- コンテナ: サーバが Running、Address が `0.0.0.0:PRIMARY_PORT`（例: `0.0.0.0:25570`）

---

## 9) 例（検証時の成功パターン）

- サーバ `mc02` を `PRIMARY_PORT=25570` で起動（Address: `0.0.0.0:25570`）。
- 外部到達テスト: `nc -vz MC_DOMAIN 25570` → `succeeded`。
- Minecraft クライアント: `MC_DOMAIN:25570` で入室可能。

---

## 10) まとめ

- DNS: `MC_DOMAIN` → `VPS_PUBLIC_IP`（DNS only）
- FRP: `frps@VPS`／`frpc@Local`、`allow_ports` と `remote_port` を一致
- Pelican: `0.0.0.0:ALLOW_RANGE` をノードに割当、サーバに `PRIMARY_PORT` を割当
- クライアント: `MC_DOMAIN:PRIMARY_PORT`（SRV 設定でポート省略も可）

この README は GitHub に公開可能なサニタイズ版です。必要に応じて、変数へ自環境の値を反映してください。

---

## 付録: 変更・復旧のコマンド集

- frps 再起動/ログ

```bash
sudo systemctl restart frps
journalctl -u frps -f
```

- frpc 再起動/ログ

```bash
sudo systemctl restart frpc
journalctl -u frpc -f
```

- 待受確認（VPS）

```bash
sudo ss -lntp | grep ${PRIMARY_PORT}
```

- 外部到達テスト

```bash
nc -vz ${MC_DOMAIN} ${PRIMARY_PORT}
```
#   P e l i c a n _ W i n g s _ F R P  
 