# Backup & deploy recipe

The concrete, copy-able artifacts for the `cloud-dev-environment` class.

## 1. backup.sh (manual, run before a reset)

```bash
#!/usr/bin/env bash
# backup.sh — run before a reset (e.g. every 30 days).
# Output: /tmp/backup-<tanggal-waktu>.tar.gz
set -euo pipefail

cd /workspaces/bravoAI

TS=$(date +%F-%H%M%S)

# Daftar global packages (untuk buat tim / reinstall nanti)
npm list -g --depth=0 > global-packages.txt 2>/dev/null || true
pip freeze > requirements.txt 2>/dev/null || true

# Tulis ke /tmp, BUKAN di dalam folder yang dibaca (avoid: "file changed as we read it")
tar -czf "/tmp/backup-${TS}.tar.gz" \
  --exclude='.git' \
  --exclude='node_modules' \
  --exclude='backup-*.tar.gz' \
  --exclude='global-packages.txt' \
  --exclude='requirements.txt' \
  . >/dev/null 2>&1 || {
    echo "✗ Gagal membuat backup."; exit 1; }

echo "✓ Backup selesai: /tmp/backup-${TS}.tar.gz"
ls -lh "/tmp/backup-${TS}.tar.gz"
```

## 2. restore.sh (rebuild from the archive)

```bash
#!/usr/bin/env bash
# restore.sh — rebuild aplikasi dari backup.
#   bash restore.sh            # cari backup-*.tar.gz di /workspaces/bravoAI
#   bash restore.sh /tmp/x.tar.gz   # pakai file tertentu
set -euo pipefail

cd /workspaces/bravoAI

BAK="${1:-backup-*.tar.gz}"

# Support globbing
if [ ! -f "$BAK" ]; then
  MATCH=$(echo backup-*.tar.gz 2>/dev/null | head -n1)
  if [ -n "$MATCH" ] && [ -e "$MATCH" ]; then
    BAK="$MATCH"
  else
    echo "✗ Belum ada file backup-*.tar.gz di /workspaces/bravoAI"
    echo "  Jalankan 'bash backup.sh' dulu, lalu tar ~ ke sini."
    exit 1
  fi
fi

echo "→ Mulai recover dari: $BAK"
tar -xzf "$BAK"

# Install dependencies sesuai package manager
if [ -f package.json ]; then
  echo "→ npm ci ..."
  npm ci --no-audit --no-fund
elif [ -f requirements.txt ]; then
  echo "→ pip install -r requirements.txt ..."
  pip install -r requirements.txt
fi

# Install package global (opsional)
if [ -f global-packages.txt ]; then
  echo "→ Install package global ..."
  while IFS= read -r pkg; do
    [ -z "$pkg" ] && continue
    case "$pkg" in \#*) continue;; esac
    npm install -g "$pkg" >/dev/null 2>&1 || true
  done < global-packages.txt
fi

echo "✓ Restore selesai."
```

## 3. GitHub Actions — monthly snapshot

`.github/workflows/backup-every-30days.yml`

```yaml
name: 30-day backup

on:
  schedule:
    # 00:00 UTC setiap 1 hari (kira-kira tiap 30 hari)
    - cron: '0 0 1 * *'
  workflow_dispatch:

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm

      - name: Install dependencies
        run: |
          if [ -f package.json ]; then
            npm ci --no-audit --no-fund
          else
            npm install --no-audit --no-fund
          fi

      - name: Capture package lists
        run: |
          npm list -g --depth=0 > global-packages.txt 2>/dev/null || true
          npm ls --depth=0 > package-tree.txt 2>/dev/null || true

      - name: Bundle snapshot into tarball
        run: |
          mkdir -p backup-snapshot
          cp -f package.json package-lock.json README.md global-packages.txt package-tree.txt backup-snapshot/ 2>/dev/null || true
          cp -r node_modules backup-snapshot/ 2>/dev/null || true
          tar -czf backup-$(date +%F).tar.gz -C backup-snapshot .

      - name: Upload backup artifact
        uses: actions/upload-artifact@v4
        with:
          name: bravoai-backup-${{ github.run_number }}
          path: backup-*.tar.gz
          retention-days: 90
```

## How to enable (what the user must provide)
1. Provide a **classic** PAT with `repo` scope: test read (`git ls-remote`) and write
   (`git push`) before scheduling — a fine-grained read-only PAT cannot schedule the
   action. (`gh auth` alone is not enough.)
2. Git identity: `git config user.name <nama>` and `git config user.email` <email>.
3. If `commit.gpgsign=true` blocks commits, disable it: `git config commit.gpgsign false`.
4. Push to origin, then confirm the action ran under Actions tab.
