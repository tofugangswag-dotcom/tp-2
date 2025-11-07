# TP 2


## 1) Préparation et configuration du webhook

- Créer un webhook sur un serveur Discord dédié aux alertes de sécurité.
- Récupérer l’URL du webhook pour l’utiliser dans les scripts d’alerte.

### Lien de mon Webhook :
```bash
https://discord.com/api/webhooks/1436421781513568306/XUV83R1feCtwSYpwTfLznOxINN_fr4J232H_2-h_PjfHNgDVdpui4V1F7leoZcqt-vUP
```

---
https://discord.com/api/webhooks/1436421781513568306/XUV83R1feCtwSYpwTfLznOxINN_fr4J232H_2-h_PjfHNgDVdpui4V1F7leoZcqt-vUP

## 2) Installer les dépendances

```bash
sudo apt update
sudo apt install -y inotify-tools curl jq
```

---

## a) Créer un fichier sensible

```bash
echo "top secret" | sudo tee /etc/secret.txt >/dev/null
sudo chmod 600 /etc/secret.txt
```


---

## b) Créer le script de surveillance

Créez le fichier `/usr/local/bin/watch-secret.sh` :

```bash
#!/usr/bin/env bash
set -u

FILE_TO_WATCH="/etc/secret.txt"
WEBHOOK_URL="https://discord.com/api/webhooks/1436421781513568306/XUV83R1feCtwSYpwTfLznOxINN_fr4J232H_2-h_PjfHNgDVdpui4V1F7leoZcqt-vUP"
HOSTNAME="$(hostname)"

command -v inotifywait >/dev/null 2>&1 || { echo "inotifywait manquant"; exit 1; }
[ -e "$FILE_TO_WATCH" ] || { echo "Fichier introuvable: $FILE_TO_WATCH"; exit 1; }

inotifywait -m -q -e open --timefmt "%Y-%m-%d %H:%M:%S" --format "%T|%e" "$FILE_TO_WATCH" | \
while IFS='|' read -r ts evt; do
  msg="🛎️ Accès détecté au fichier sensible:
- Fichier : $FILE_TO_WATCH
- Événement : $evt
- Hôte : $HOSTNAME
- Date : $ts"

  curl -sS -H "Content-Type: application/json" \
       -X POST -d "$(jq -n --arg c "$msg" '{content:$c}')" \
       "$WEBHOOK_URL" >/dev/null
done
```

je le rends-le exécutable :
```bash
sudo chmod +x /usr/local/bin/watch-secret.sh
```

---

## c) Lancer le script manuellement

```bash
sudo /usr/local/bin/watch-secret.sh
```

Puis, dans un autre terminal :
```bash
sudo cat /etc/secret.txt >/dev/null
```


Crée le script `/usr/local/bin/watch-ssh-hours.sh` :

```bash
sudo tee /usr/local/bin/watch-ssh-hours.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -u

# --- Paramètres ---
START_HOUR=9          # inclus
END_HOUR=18           # exclus (>=18 = hors horaires)
WEBHOOK_URL="https://discord.com/api/webhooks/1436421781513568306/XUV83R1feCtwSYpwTfLznOxINN_fr4J232H_2-h_PjfHNgDVdpui4V1F7leoZcqt-vUP"

HOSTNAME="$(hostname)"
TZ_LOCAL="$(date +%Z)"
COOLDOWN=300          # anti-spam (secondes) par user@ip:method
declare -A LAST_ALERT
DEBUG=1               # 0 pour silencieux

log(){ [ "$DEBUG" -eq 1 ] && echo "[watch-ssh-hours] $*"; }

send_alert () {
  local msg="$1"
  curl -sS -H "Content-Type: application/json"        -X POST -d "{"content":"${msg//"/\"}"}"        "$WEBHOOK_URL" >/dev/null
}

handle_login () {
  local user="$1" ip="$2" method="$3"
  local hour now_epoch
  hour="$(date +%H)"
  now_epoch="$(date +%s)"

  if (( 10#$hour < START_HOUR || 10#$hour >= END_HOUR )); then
    local key="${user}@${ip}:${method}"
    local last="${LAST_ALERT[$key]:-0}"
    if (( now_epoch - last >= COOLDOWN )); then
      LAST_ALERT[$key]=$now_epoch
      local ts="$(date '+%Y-%m-%d %H:%M:%S')"
      local msg="🚨 Connexion SSH HORS horaires
- Hôte : $HOSTNAME
- Utilisateur : $user
- Adresse : $ip
- Méthode : $method
- Heure locale : $ts ($TZ_LOCAL)
- Plage autorisée : ${START_HOUR}h00–${END_HOUR}h00"
      log "ALERTE: $user@$ip ($method)"
      send_alert "$msg"
    else
      log "Cooldown actif pour $key"
    fi
  else
    log "Connexion dans la plage horaire (heure=$hour)"
  fi
}

log "Démarrage (heures ouvrées ${START_HOUR}–${END_HOUR}) sur $HOSTNAME"

if command -v journalctl >/dev/null 2>&1; then
  journalctl -f -u ssh -u sshd -o cat |   while IFS= read -r line; do
    if [[ "$line" =~ Accepted[[:space:]]+([A-Za-z0-9_-]+)[[:space:]]+for[[:space:]]+([^[:space:]]+)[[:space:]]+from[[:space:]]+([^[:space:]]+) ]]; then
      method="${BASH_REMATCH[1]}"; user="${BASH_REMATCH[2]}"; ip="${BASH_REMATCH[3]}"
      log "match journalctl: user=$user ip=$ip method=$method"
      handle_login "$user" "$ip" "$method"
    fi
  done
else
  tail -F /var/log/auth.log |   while IFS= read -r line; do
    if [[ "$line" =~ sshd\[.*\]:[[:space:]]Accepted[[:space:]]+([A-Za-z0-9_-]+)[[:space:]]+for[[:space:]]+([^[:space:]]+)[[:space:]]+from[[:space:]]+([^[:space:]]+) ]]; then
      method="${BASH_REMATCH[1]}"; user="${BASH_REMATCH[2]}"; ip="${BASH_REMATCH[3]}"
      log "match auth.log: user=$user ip=$ip method=$method"
      handle_login "$user" "$ip" "$method"
    fi
  done
fi
EOF

sudo chmod +x /usr/local/bin/watch-ssh-hours.sh
```

---

## service systemd

```bash
sudo tee /etc/systemd/system/watch-ssh-hours.service >/dev/null <<'EOF'
[Unit]
Description=Surveillance des connexions SSH hors horaires de bureau (Discord)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/watch-ssh-hours.sh
Restart=always
RestartSec=3
User=root
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now watch-ssh-hours.service
sudo systemctl status watch-ssh-hours.service --no-pager
```

---

## 3. Surveillance des connexions SSH hors des horaires de bureau

## Étape 1 : script de surveillance

Crée le script `/usr/local/bin/watch-ssh-hours.sh` :

```bash
sudo tee /usr/local/bin/watch-ssh-hours.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -u

# --- Paramètres ---
START_HOUR=9          # inclus
END_HOUR=18           # exclus (>=18 = hors horaires)
WEBHOOK_URL="https://discord.com/api/webhooks/1436421781513568306/XUV83R1feCtwSYpwTfLznOxINN_fr4J232H_2-h_PjfHNgDVdpui4V1F7leoZcqt-vUP"

HOSTNAME="$(hostname)"
TZ_LOCAL="$(date +%Z)"
COOLDOWN=300          # anti-spam (secondes) par user@ip:method
declare -A LAST_ALERT
DEBUG=1               # 0 pour silencieux

log(){ [ "$DEBUG" -eq 1 ] && echo "[watch-ssh-hours] $*"; }

send_alert () {
  local msg="$1"
  curl -sS -H "Content-Type: application/json"        -X POST -d "{"content":"${msg//"/\"}"}"        "$WEBHOOK_URL" >/dev/null
}

handle_login () {
  local user="$1" ip="$2" method="$3"
  local hour now_epoch
  hour="$(date +%H)"
  now_epoch="$(date +%s)"

  if (( 10#$hour < START_HOUR || 10#$hour >= END_HOUR )); then
    local key="${user}@${ip}:${method}"
    local last="${LAST_ALERT[$key]:-0}"
    if (( now_epoch - last >= COOLDOWN )); then
      LAST_ALERT[$key]=$now_epoch
      local ts="$(date '+%Y-%m-%d %H:%M:%S')"
      local msg="🚨 Connexion SSH HORS horaires
- Hôte : $HOSTNAME
- Utilisateur : $user
- Adresse : $ip
- Méthode : $method
- Heure locale : $ts ($TZ_LOCAL)
- Plage autorisée : ${START_HOUR}h00–${END_HOUR}h00"
      log "ALERTE: $user@$ip ($method)"
      send_alert "$msg"
    else
      log "Cooldown actif pour $key"
    fi
  else
    log "Connexion dans la plage horaire (heure=$hour)"
  fi
}

log "Démarrage (heures ouvrées ${START_HOUR}–${END_HOUR}) sur $HOSTNAME"

if command -v journalctl >/dev/null 2>&1; then
  journalctl -f -u ssh -u sshd -o cat |   while IFS= read -r line; do
    if [[ "$line" =~ Accepted[[:space:]]+([A-Za-z0-9_-]+)[[:space:]]+for[[:space:]]+([^[:space:]]+)[[:space:]]+from[[:space:]]+([^[:space:]]+) ]]; then
      method="${BASH_REMATCH[1]}"; user="${BASH_REMATCH[2]}"; ip="${BASH_REMATCH[3]}"
      log "match journalctl: user=$user ip=$ip method=$method"
      handle_login "$user" "$ip" "$method"
    fi
  done
else
  tail -F /var/log/auth.log |   while IFS= read -r line; do
    if [[ "$line" =~ sshd\[.*\]:[[:space:]]Accepted[[:space:]]+([A-Za-z0-9_-]+)[[:space:]]+for[[:space:]]+([^[:space:]]+)[[:space:]]+from[[:space:]]+([^[:space:]]+) ]]; then
      method="${BASH_REMATCH[1]}"; user="${BASH_REMATCH[2]}"; ip="${BASH_REMATCH[3]}"
      log "match auth.log: user=$user ip=$ip method=$method"
      handle_login "$user" "$ip" "$method"
    fi
  done
fi
EOF

sudo chmod +x /usr/local/bin/watch-ssh-hours.sh
```

---

## Étape 2 : service systemd

```bash
sudo tee /etc/systemd/system/watch-ssh-hours.service >/dev/null <<'EOF'
[Unit]
Description=Surveillance des connexions SSH hors horaires de bureau (Discord)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/watch-ssh-hours.sh
Restart=always
RestartSec=3
User=root
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now watch-ssh-hours.service
sudo systemctl status watch-ssh-hours.service --no-pager
```

---

### Forcer une alerte
- Je me connecte-toi en SSH **hors 9–18**  


---

### 4) Automatisation avec cron pour une surveillance continue

## a) Éditer le crontab **root**

les scripts lisent des journaux/chemins protégés : on programme le cron du super‑utilisateur.

```bash
sudo crontab -e
```

Ajoute **à la fin** :

```cron
# --- Surveillance continue du fichier sensible au démarrage ---
@reboot /usr/local/bin/watch-secret.sh >> /var/log/watch-secret.log 2>&1 &

# --- Connexions SSH hors horaires (toutes les 5 minutes) ---
# flock évite les doublons si une exécution dure > 5 min
*/5 * * * * flock -n /run/watch-ssh-hours.lock /usr/local/bin/watch-ssh-hours.sh >> /var/log/watch-ssh-hours.log 2>&1
```

> - `@reboot` démarre la surveillance du fichier **à chaque boot**.
> - `flock -n` empêche un nouveau lancement si l’instance précédente tourne encore.
> - La redirection `>> … 2>&1` écrit les sorties dans des **logs dédiés**.

---
