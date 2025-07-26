# 🛡️ Auto-Ban SSH Brute Force su Raspberry Pi 4 con ipset + iptables + Telegram Bot

Questa guida ti permette di rilevare tentativi SSH falliti e bloccare automaticamente gli IP sospetti usando `ipset`, `iptables`, e uno script personalizzato con **notifiche Telegram Bot** solo per **i nuovi IP** bannati, evitando spam o elenchi lunghi.

---

## 📋 Requisiti

- Raspberry Pi con Raspberry OS / Debian / Ubuntu
- SSH attivo
- `systemd` + `journald` per logging
- Accesso root
- Token Telegram Bot + ID chat

---

## 1️⃣ Installa ipset e iptables-persistent

```bash
sudo apt update
sudo apt install ipset iptables-persistent curl
````

---

## 2️⃣ Crea e attiva la blacklist ipset

```bash
sudo ipset create ssh_blacklist hash:ip
sudo iptables -I INPUT -m set --match-set ssh_blacklist src -j DROP
```

---

## 3️⃣ Crea Telegram Bot

1. Apri Telegram, cerca `@BotFather`
2. Invia `/newbot` e segui le istruzioni
3. Otterrai un **TOKEN** → salvalo
4. In un’altra chat (gruppo o privata), invia un messaggio qualsiasi
5. Vai su `https://api.telegram.org/bot<TOKEN>/getUpdates`
6. Trova il tuo **chat\_id** (es: `123456789`)

---

## 4️⃣ Crea lo script `/usr/local/bin/ssh-ban.sh`

```bash
sudo nano /usr/local/bin/ssh-ban.sh
```

Incolla:

```bash
#!/bin/bash

# --- Configurazione Telegram ---
TELEGRAM_BOT_TOKEN="INSERISCI_IL_TUO_TOKEN"
TELEGRAM_CHAT_ID="INSERISCI_LA_TUA_CHATID"
SOGLIA=5
TMPFILE="/tmp/ssh_failed_ips.txt"
LOGFILE="/var/log/ssh-ban.log"

# --- Estrai IP con più di SOGLIA tentativi falliti ---
journalctl _COMM=sshd | grep "Failed password" | \
awk '{for(i=1;i<=NF;i++) if ($i ~ /^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$/) print $i}' | \
sort | uniq -c | awk -v soglia="$SOGLIA" '$1 >= soglia {print $2}' > "$TMPFILE"

# --- Gestione ban e notifica ---
while read ip; do
  if ! ipset test ssh_blacklist "$ip" &>/dev/null; then
    ipset add ssh_blacklist "$ip"
    logger "[SSH-BAN] IP bannato: $ip"
    echo "$(date '+%F %T') BANNED: $ip" >> "$LOGFILE"

    # Notifica Telegram
    MSG="🚫 Nuovo IP bannato per brute force SSH:\n$ip"
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d chat_id="${TELEGRAM_CHAT_ID}" \
      -d text="$MSG" \
      -d parse_mode="Markdown" > /dev/null
  fi
done < "$TMPFILE"

rm -f "$TMPFILE"
```

Rendi eseguibile:

```bash
sudo chmod +x /usr/local/bin/ssh-ban.sh
```

---

## 5️⃣ Automatizza con `cron`

```bash
sudo crontab -e
```

Aggiungi:

```cron
*/15 * * * * /usr/local/bin/ssh-ban.sh
```

---

## 6️⃣ Salva regole iptables e ipset

```bash
sudo netfilter-persistent save
sudo ipset save > /etc/ipset.conf
```

---

## 7️⃣ Ripristina ipset al boot

```bash
sudo nano /etc/systemd/system/ipset-restore.service
```

Contenuto:

```ini
[Unit]
Description=Restore ipset rules
Before=netfilter-persistent.service
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ipset restore < /etc/ipset.conf
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Abilita:

```bash
sudo systemctl daemon-reexec
sudo systemctl enable ipset-restore.service
```

---

## ✅ Verifiche

Visualizza IP bannati:

```bash
sudo ipset list ssh_blacklist
```

Controlla log personalizzati:

```bash
cat /var/log/ssh-ban.log
```

Controlla regole iptables:

```bash
sudo iptables -L INPUT -v -n
```

---

## 📌 Note

* La soglia è impostata a **5 tentativi** per IP → modificabile nello script
* Il log locale si trova in `/var/log/ssh-ban.log`
* Solo i **nuovi IP** bannati generano notifica Telegram
* Niente spam all’avvio o dopo reboot

---

## 📎 File utili

| Percorso                                    | Scopo                          |
| ------------------------------------------- | ------------------------------ |
| `/usr/local/bin/ssh-ban.sh`                 | Script di analisi e blocco IP  |
| `/etc/ipset.conf`                           | Backup lista ipset persistente |
| `/etc/systemd/system/ipset-restore.service` | Restore ipset al boot          |
| `/var/log/ssh-ban.log`                      | Log ban manuali e cron         |

---

## 🔐 Consigli finali

* Usa `iptables -C` per evitare duplicati se vuoi ottimizzare ulteriormente
* Puoi estendere lo script per loggare anche `Accepted password` o login sospetti
* Se vuoi notifiche anche via mail o su syslog remoto, basta estendere lo script
