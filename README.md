# System Status Monitor

Ubuntu/Linux ターミナル向けの軽量システムステータスモニターです。
重い監視ツール（htop等）を入れることなく、標準的なShellコマンドだけでシステムの健康状態、OOM（メモリ不足）の予兆、全物理ディスクの容量などをリアルタイムで可視化します。

## 特徴 (Features)

* **ハイブリッド更新:** 時計は **1秒**、データ取得は **10秒**（設定可能）のデュアル更新。
* **OOM予兆検知:** メモリ不足やSwap枯渇、PSI (Pressure Stall Information) を監視。
* **フリッカーフリー:** `tput` 制御により、画面のチラつきがありません。
* **全ディスク自動検出:** 物理ディスクを自動判別して一覧表示します。

## インストール手順 (Installation)

以下のコマンドブロックをターミナルに貼り付けて実行してください。
**「スクリプトの作成」**と**「ログイン時の自動起動設定」**を一括で完了します。
以下のファイル、フォルダが自動生成されます。
- ~/monitor.sh
- ~/.config/autostart/
- ~/.config/autostart/monitor.desktop

```bash
# 1. スクリプトの作成
cat << 'EOF' > ~/monitor.sh
#!/bin/bash
TARGET_DISK="${1:-ALL}"; INTERVAL="${2:-10}"; count=0; cached_output=""; script_name="monitor.sh"
EL=$(tput el); ED=$(tput ed); clear
while true; do
    cols=$(tput cols)
    if [ $count -le 0 ]; then
        raw_data=$(
            echo "=== NETWORK (Interface : IP) ==="; ip -4 -br addr show | grep -v "^lo" | awk '{printf "%-10s %s\n", $1, $3}'; echo
            echo "=== CPU USAGE ==="; top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1"% (Used)"}'; echo
            if [ "$TARGET_DISK" = "ALL" ]; then echo "=== DISK USAGE (All Physical) ==="; df -hP | grep -vE '^tmpfs|^devtmpfs|^loop|^overlay|^none|^udev'; else echo "=== DISK USAGE ($TARGET_DISK) ==="; df -hP "$TARGET_DISK" 2>/dev/null | head -1; df -hP "$TARGET_DISK" 2>/dev/null | tail -1; fi; echo
            echo "=== POWER ==="; upower -i $(upower -e | grep BAT) 2>/dev/null | grep -E "energy-rate|percentage|state" || echo "No Battery Sensor"; echo
            echo "=== MEMORY & OOM CHECK ==="; free -h; echo
            echo "[OOM Risk Analysis]"; AVL=$(grep MemAvailable /proc/meminfo | awk '{print $2}'); SWP=$(grep SwapFree /proc/meminfo | awk '{print $2}'); PSI=$(grep "some" /proc/pressure/memory 2>/dev/null | awk -F"avg10=" '{print $2}' | awk '{print $1}' || echo "0.00")
            if [ "$AVL" -lt 500000 ]; then echo "🚨 DANGER: Memory Critical!"; elif [ "$SWP" -lt 500000 ] && [ "$SWP" -gt 0 ]; then echo "⚠️ WARNING: Swap filling up!"; else echo "✅ Capacity: Safe"; fi
            if [ -n "$PSI" ]; then echo "Memory Pressure (10s avg): $PSI%"; fi
        )
        cached_output=$(echo "$raw_data" | sed "s/$/${EL}/"); count=$INTERVAL
    fi
    tput cup 0 0
    printf '%*s\n' "$cols" '' | tr ' ' '='; left_info="$(date '+%H:%M:%S') | $(whoami)@$(hostname)"; right_info="Next Update: ${count}s "
    pad_len=$(( cols - ${#left_info} - ${#right_info} )); [ $pad_len -lt 1 ] && pad_len=1
    printf "%s%*s%s${EL}\n" "$left_info" "$pad_len" "" "$right_info"; printf '%*s\n' "$cols" '' | tr ' ' '='; echo "${EL}"; echo "$cached_output"; echo "${EL}"
    printf '%*s\n' "$cols" '' | tr ' ' '='; echo " Usage: ./$script_name [ALL|Path] [SEC] (e.g. ./$script_name ALL 5) | Ctrl+C to Exit${EL}"; printf '%*s' "$cols" '' | tr ' ' '='; tput ed
    count=$((count - 1)); sleep 1
done
EOF
chmod +x ~/monitor.sh

# 2. 自動起動設定 (gnome-terminalで最大化して起動)
mkdir -p ~/.config/autostart
cat << EOF > ~/.config/autostart/monitor.desktop
[Desktop Entry]
Type=Application
Exec=gnome-terminal --maximize -- bash -c "$HOME/monitor.sh"
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=System Monitor
Comment=Start system monitor on login
EOF

echo "✅ インストール完了: 次回ログイン時から自動起動します。"
```

## 使い方 (Usage)

### 手動起動
```bash
./monitor.sh
```

### カスタマイズ起動
引数を指定することで、監視対象や更新頻度を変更できます。
※自動起動時の設定を変えたい場合は、`~/.config/autostart/monitor.desktop` 内の `Exec` 行を書き換えてください。

```bash
# 構文: ./monitor.sh [監視対象] [秒数]

# 例1: 1秒ごとに超高速更新
./monitor.sh ALL 1

# 例2: /home ディレクトリだけを監視
./monitor.sh /home
```

## アンインストール (Uninstall)
不要になった場合は以下のコマンドで削除してください。

```bash
rm ~/monitor.sh
rm ~/.config/autostart/monitor.desktop
```
