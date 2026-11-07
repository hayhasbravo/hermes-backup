# hermes-backup

Backup snapshots dari **`.hermes agent`** di codespace kamu.

Tujuannya: biar nggak rugi konfigurasi, skill, kanban, logs, dan memory kalau codespace lama reset/expired.

File penting yang disimpan di repo ini:
- `config.yaml` — konfigurasi utama Hermes
- `SOUL.md`, `USER.md`, `MEMORY.md` — identitas & memori pribadi
- `.skills_prompt_snapshot.json`, `channel_directory.json` — snapshot skills & channel
- `skills/` — semua skill template yang kamu pakai
- `kanban.db` — isi kanban
- `memories/` — memori pribadi
- `logs/` — log aktivitas
- `backups/config/` — snapshot konfigurasi sebelumnya

> 🔒 **TIDAK ikut:** `.env`, `auth.json`, caches (`cache/`, `image_cache/`, `audio_cache/`), `installs/`, `hermes-agent/`, `lsp/`, dan lock-file SQLite (`kanban.db-wal`, `*.db-shm`).  
> Kharismanya: `.env`, `auth.json` hanya boleh kamu simpan sendiri (password manager / secret).

---

## 📥 Cara Pasang (1 menit)

```bash
# 1. clone repo
git clone https://github.com/hayhasbravo/hermes-backup.git
cd hermes-backup

# 2. salin file penting ke /root/.hermes
cp config.yaml SOUL.md USER.md MEMORY.md .skills_prompt_snapshot.json channel_directory.json /
cp -r skills memories kanban.db logs backups/config /

# 3. (opsional) pastikan webhook/hook tersedia
ls scripts/
```

Setelah itu, restart proses hermes:

```bash
hermes restart
```

> 🔁 Karena `kanban.db`, `logs/`, dan `backups/` punya file lock internal (SQLite), jangan copy saat hermes masih berjalan — hentikan dulu, lalu restore.

---

## 🔄 Cara Update Tiap 30 Hari

```bash
cd hermes-backup
bash backup.sh
```

Script otomatis:
1. Sync konfigurasi penting ke repo
2. Commit & push ke GitHub

> Kalau kamu pakai settingan tersembunyi, jalankan dulu di sesi aktif, baru push.

---

## 🗂️ Struktur Repo

```
hermes-backup/
├── .gitignore          # eksklusi secret & cache
├── backup.sh           # script auto backup
├── config.yaml         # konfigurasi Hermes
├── .skills_prompt_snapshot.json
├── SOUL.md
├── USER.md
├── MEMORY.md
├── channel_directory.json
├── skills/             # semua skill kamu
├── memories/           # memori pribadi
├── kanban.db           # isi kanban (rekomendasi tampilkan saat restore)
├── logs/               # log hermes
└── backups/config/     # snapshot config lama
```

---

## ⚠️ Tips & Catatan

1. **Jangan** taruh `.env` / `auth.json` di repo ini.
2. Backup otomatis diusahakan saat codespace **tidak sedang sibuk**, atau pakai GitHub Actions.
3. `kanban.db` dan `logs/` berisi data runtime, jangan salin sambil hermes hidup.

---

## ❓ Troubleshoot

- **`push denied` lintas login** → Lakukan `gh auth login` dan pastikan scope `repo` aktif.
- **Commit gagal** → Jalankan `git config user.name` dan `git config user.email`.
- **Restore error SQLite** → Hentikan hermes dulu, clean `kanban.db-wal`, `-shm`, lanjut restore.
