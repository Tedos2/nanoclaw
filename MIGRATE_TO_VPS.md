# VPS Migration Checklist — Ted's Second Brain

When local-on-Mac is proven and you want 24/7 uptime, run this playbook on a fresh Ubuntu 24.04 VPS (Hetzner CX22 / DO droplet).

## What lives OUTSIDE git (gitignored)

The public fork `Tedos2/nanoclaw` only ships upstream code + the whatsapp skill merge. The **personal state** lives in gitignored paths and must be copied by hand:

| Path | Contains | Portable? |
|---|---|---|
| `.env` | `ASSISTANT_NAME`, `TZ`, `ONECLI_URL` | Yes — re-create on VPS |
| `store/auth/` | WhatsApp Baileys session | **No** — pairing-code re-link required |
| `store/messages.db` | SQLite: chats, registered_groups, messages | Yes — scp preserves |
| `groups/whatsapp_main/CLAUDE.md` | TedrosAi persona, Hebrew prefs, Ted facts | Yes — scp |
| `groups/global/CLAUDE.md`, `groups/main/CLAUDE.md` | Default personas | Yes — already in fork |
| `vault/` (self-versioned git repo) | Obsidian second brain | Yes — push to private git remote and re-clone |
| OneCLI secrets (Anthropic token) | Claude Max OAuth | **No** — re-create via `claude setup-token` |

## Steps on the VPS

### 1. Prep
```bash
apt update && apt install -y git curl build-essential
curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && apt install -y nodejs
curl -fsSL https://get.docker.com | sh && systemctl enable --now docker
usermod -aG docker $USER   # log out+in
```

### 2. Clone fork + bootstrap
```bash
gh auth login   # or use a PAT
git clone https://github.com/Tedos2/nanoclaw.git ~/NanoTed
cd ~/NanoTed
bash setup.sh
```

### 3. Install & register OneCLI
```bash
curl -fsSL onecli.sh/install | sh
curl -fsSL onecli.sh/cli/install | sh
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

### 4. Inject Claude Max token
On your Mac, run `claude setup-token`, copy the token, then on VPS:
```bash
onecli secrets create \
  --name "Anthropic Token" --type anthropic \
  --host-pattern api.anthropic.com \
  --value 'PASTE_TOKEN_HERE'
```

### 5. Copy local state from Mac
From Mac:
```bash
rsync -avz --exclude node_modules --exclude dist --exclude .git \
  ~/NanoTed/groups/whatsapp_main/ \
  VPS:~/NanoTed/groups/whatsapp_main/
rsync -avz ~/NanoTed/store/messages.db VPS:~/NanoTed/store/
rsync -avz ~/NanoTed/vault/ VPS:~/NanoTed/vault/     # or git push/pull
echo "TZ=Asia/Jerusalem
ONECLI_URL=http://localhost:10254
ASSISTANT_NAME=\"TedrosAi\"" | ssh VPS 'cat > ~/NanoTed/.env'
```

**Do NOT copy `store/auth/`** — Baileys sessions are tied to the device. Re-link instead (next step).

### 6. Re-link WhatsApp (pairing code)
```bash
cd ~/NanoTed
npm run auth    # prints pairing code
```
On your phone: WhatsApp → Settings → Linked devices → Link with phone number → enter code.
This writes a new `store/auth/` on the VPS and un-links the Mac device (that's fine).

### 7. Start service
```bash
npx tsx setup/index.ts --step service    # creates systemd user unit
systemctl --user enable --now nanoclaw
```

### 8. Verify
```bash
systemctl --user status nanoclaw
journalctl --user -u nanoclaw -f
```

Send a WhatsApp message to your self-chat — TedrosAi should reply within 5s.

### 9. Firewall
```bash
ufw allow 22/tcp
ufw allow from YOUR_MAC_IP to any port 10254    # OneCLI dashboard, IP-restricted
ufw --force enable
```

### 10. Decommission Mac (optional)
Only after VPS is proven stable:
```bash
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```
WhatsApp device link will have moved to VPS automatically at step 6.

## Rollback
If the VPS acts up and you need local again:
```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
```
Re-run `npm run auth` on Mac to re-link WhatsApp.

## Kill switch (any time)
- Stop the brain: `systemctl --user stop nanoclaw` (VPS) or `launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist` (Mac)
- Cut the WhatsApp wire: WhatsApp → Settings → Linked devices → log out the NanoClaw device
- Revoke Claude token: https://claude.com/settings/tokens → revoke

## Backup cadence
Add a cron on the VPS:
```cron
0 3 * * * tar czf ~/backups/nanoted-$(date +\%F).tgz ~/NanoTed/vault ~/NanoTed/groups/whatsapp_main ~/NanoTed/store/messages.db ~/NanoTed/.env && find ~/backups -mtime +30 -delete
```
