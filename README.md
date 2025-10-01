sudo tee /usr/bin/stop-acero >/dev/null <<'EOF'
#!/usr/bin/env bash
# Stop AceRO servers
set -Eeuo pipefail
DIR="/root/AceRO2024Trunk"
target="${1:-all}"

case "$target" in
  all)   svcs=(login char web mapdbg) ;;
  map)   svcs=(mapdbg) ;;
  login) svcs=(login) ;;
  char)  svcs=(char) ;;
  web)   svcs=(web) ;;
  *) echo "Usage: stop-acero [all|map|login|char|web]"; exit 2 ;;
esac

# stop screens
for s in "${svcs[@]}"; do
  screen -S "$s" -X quit >/dev/null 2>&1 || true
done

# ensure processes die
kill_term() { pkill -TERM -f "$1" >/dev/null 2>&1 || true; }
kill_kill() { pkill -KILL -f "$1" >/dev/null 2>&1 || true; }

sleep 1
if [[ "$target" == "map" ]]; then
  kill_term "gdb.*map-server"
  kill_term "$DIR/map-server"
  sleep 2
  pgrep -f "gdb.*map-server" >/dev/null && kill_kill "gdb.*map-server"
  pgrep -f "$DIR/map-server"  >/dev/null && kill_kill "$DIR/map-server"
else
  for pat in "$DIR/login-server" "$DIR/char-server" "$DIR/web-server" "$DIR/map-server" "gdb.*map-server"; do
    kill_term "$pat"
  done
  sleep 2
  for pat in "$DIR/login-server" "$DIR/char-server" "$DIR/web-server" "$DIR/map-server" "gdb.*map-server"; do
    pgrep -f "$pat" >/dev/null && kill_kill "$pat"
  done
fi

echo "Stopped. Check: screen -ls"
EOF
sudo chmod +x /usr/bin/stop-acero
