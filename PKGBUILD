# Maintainer: You <you@example.com>
pkgname=keyboard-chattering-fix
pkgver=0.0.1
pkgrel=1
pkgdesc="Filter mechanical keyboard chattering system-wide"
arch=('x86_64')
url="https://github.com/w2sv/KeyboardChatteringFix-Linux"
license=('MIT')
depends=('python' 'libevdev')
makedepends=('git' 'python-pip')
source=("git+https://github.com/w2sv/KeyboardChatteringFix-Linux.git")
sha256sums=('SKIP')

post_install() {
  cat >&2 <<'EOF'

==================================================
  Keyboard Chattering Fix installed successfully!
==================================================

To run the service, enable it for system boot:

  sudo systemctl daemon-reload
  sudo systemctl enable --now chattering_fix.service

The service has been configured with your selected keyboard and threshold.
EOF
}

pre_remove() {
  if command -v systemctl >/dev/null 2>&1; then
    if systemctl is-enabled --quiet chattering_fix.service >/dev/null 2>&1; then
      echo "Disabling and stopping chattering_fix.service..." >&2
      systemctl disable --now chattering_fix.service >/dev/null 2>&1 || true
    fi
  fi
}

post_upgrade() {
  echo "chattering_fix package was upgraded. Restart the service if it's running:" >&2
  echo "  sudo systemctl daemon-reload" >&2
  echo "  sudo systemctl restart chattering_fix.service" >&2
}

build() {
  # No build step required; all sources are embedded in this PKGBUILD
  :
}

package() {
  cd "$srcdir/KeyboardChatteringFix-Linux"

  install -d "$pkgdir/usr/lib/$pkgname"
  # Install the python source under /usr/lib/<pkgname>/ so the wrapper can set PYTHONPATH
  cp -a src "$pkgdir/usr/lib/$pkgname/"

  # Create a virtualenv under the package and install libevdev into it
  install -d "$pkgdir/usr/lib/$pkgname/venv"
  /usr/bin/python3 -m venv "$pkgdir/usr/lib/$pkgname/venv"
  # Upgrade pip inside venv and install runtime Python deps
  "$pkgdir/usr/lib/$pkgname/venv/bin/python" -m pip install --upgrade pip
  "$pkgdir/usr/lib/$pkgname/venv/bin/pip" install libevdev~=0.11 inquirer

  # Wrapper script that uses the venv python and includes package source in PYTHONPATH
  install -d "$pkgdir/usr/bin"
  cat > "$pkgdir/usr/bin/keyboard-chattering-fix" <<'EOF'
#!/bin/sh
# Run the packaged keyboard chattering fixer using bundled virtualenv
exec env PYTHONPATH=/usr/lib/keyboard-chattering-fix /usr/lib/keyboard-chattering-fix/venv/bin/python -m src "$@"
EOF
  chmod 755 "$pkgdir/usr/bin/keyboard-chattering-fix"

  # Install an example chattering_fix.sh helper (generated so the PKGBUILD is standalone)
  install -d "$pkgdir/usr/lib/$pkgname/examples"
  cat > "$pkgdir/usr/lib/$pkgname/examples/chattering_fix.sh" <<'EOF'
#!/usr/bin/env bash
# Example wrapper to run the installed keyboard chattering fix package.
# Edit KEYBOARD and THRESHOLD as needed.
KEYBOARD=""   # e.g. /dev/input/by-id/usb-FOO-event-kbd
THRESHOLD=30

cd /usr/lib/keyboard-chattering-fix || exit 1
if [ -n "$KEYBOARD" ]; then
  exec /usr/bin/keyboard-chattering-fix -k "$KEYBOARD" -t "$THRESHOLD"
else
  exec /usr/bin/keyboard-chattering-fix -t "$THRESHOLD"
fi
EOF
  chmod 755 "$pkgdir/usr/lib/$pkgname/examples/chattering_fix.sh"

  # Install systemd unit. Prompt the packager to select a default keyboard (optional).
  install -d "$pkgdir/usr/lib/systemd/system"

  # Detect candidate keyboards under /dev/input/by-id/*-event-kbd
  kbd_candidates=(/dev/input/by-id/*-event-kbd)
  if [ ! -e "${kbd_candidates[0]}" ]; then
    echo "No keyboards detected under /dev/input/by-id/*-event-kbd; service will auto-detect at runtime."
    selected_kbd=""
  else
    echo "Detected keyboards:"
    idx=1
    for k in "${kbd_candidates[@]}"; do
      printf "  %d) %s\n" "$idx" "$k"
      idx=$((idx+1))
    done
    printf "Select keyboard number to set as service default (ENTER to skip): "
    read sel || sel=""
    if [ -n "$sel" ] && printf "%s" "$sel" | grep -qE '^[0-9]+$' && [ "$sel" -ge 1 ] && [ "$sel" -le "${#kbd_candidates[@]}" ]; then
      selected_kbd="${kbd_candidates[$((sel-1))]}"
      echo "Selected: $selected_kbd"
    else
      selected_kbd=""
      echo "No selection. Service will auto-detect at runtime."
    fi
  fi

  # Prompt for threshold in milliseconds (optional; default 30). Can be set non-interactively with PKG_THRESHOLD.
  if [ -n "$PKG_THRESHOLD" ]; then
    threshold="$PKG_THRESHOLD"
    echo "Using non-interactive PKG_THRESHOLD=$threshold"
  else
    printf "Enter threshold in ms (ENTER for default 30): "
    read threshold || threshold=""
    if [ -z "$threshold" ] || ! printf "%s" "$threshold" | grep -qE '^[0-9]+$'; then
      echo "No valid threshold entered; using default 30"
      threshold=30
    else
      echo "Selected threshold: $threshold ms"
    fi
  fi

  execstart="/usr/bin/keyboard-chattering-fix -t $threshold"
  if [ -n "$selected_kbd" ]; then
    execstart="$execstart -k $selected_kbd"
  fi

  cat > "$pkgdir/usr/lib/systemd/system/chattering_fix.service" <<EOF
[Unit]
Description=Keyboard Chattering Fix service

[Service]
# The program will try to auto-detect your keyboard if no -k option is supplied
ExecStart=$execstart
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
  chmod 644 "$pkgdir/usr/lib/systemd/system/chattering_fix.service"

  # Documentation
  install -d "$pkgdir/usr/share/doc/$pkgname"
  install -m644 LICENSE README.md "$pkgdir/usr/share/doc/$pkgname/"
}

