# 🛡️ Auto-Ban SSH con ipset + iptables + Telegram Bot (Efficient Version)

Questa guida implementa un sistema automatico di protezione contro attacchi brute-force SSH usando `ipset` e `iptables`, con notifiche via Telegram e parsing incrementale dei log `journalctl` per ridurre il carico sul Raspberry Pi.

> ✅ **Supporta notifiche Telegram solo per IP realmente nuovi**
> ✅ **timestamp persistente per tracciamento (salvato su file)**
> ✅ **Analizza solo i log recenti dalla precedente esecuzione (`journalctl --since`)**
> ✅ **notifiche solo sui nuovi IP realmente aggiunti**
> ✅ **Persistente al boot**

---

## 📋 Requisiti

- Raspberry Pi con Linux (Debian-based)
- SSH attivo e loggato tramite `journalctl`
- Telegram Bot e chat ID configurati
- `ipset`, `iptables`, `curl` installati

---

## 1️⃣ Installa pacchetti

```bash
sudo apt update
sudo apt install ipset iptables-persistent curl
````

---

## 2️⃣ Configura Telegram Bot

Crea un bot tramite `@BotFather`, ottieni il `BOT_TOKEN` e `CHAT_ID`.

---

## 3️⃣ Crea lista ipset

```bash
sudo ipset create ssh_blacklist hash:ip
```

Aggiungi la regola iptables:

```bash
sudo iptables -I INPUT -m set --match-set ssh_blacklist src -j DROP
```

---

## 4️⃣ Crea script incrementale `/usr/local/bin/ssh-ban.sh`

```bash
sudo vi /usr/local/bin/ssh-ban.sh
```

Incolla:

```bash
#!/bin/bash

# --- CONFIG ---
SOGLIA=5
TELEGRAM_BOT_TOKEN="INSERISCI_IL_TUO_TOKEN"
TELEGRAM_CHAT_ID="INSERISCI_LA_TUA_CHATID"
TIMESTAMP_FILE="/var/lib/ssh-ban/last_check.timestamp"
LOGFILE="/var/log/ssh-ban.log"
mkdir -p "$(dirname "$TIMESTAMP_FILE")"

# --- Calcola orario da cui analizzare ---
if [ -f "$TIMESTAMP_FILE" ]; then
    SINCE=$(cat "$TIMESTAMP_FILE")
else
    SINCE="1 hour ago"
fi

# --- Aggiorna timestamp subito per evitare salti in caso di crash ---
date -u +"%Y-%m-%d %H:%M:%S" > "$TIMESTAMP_FILE"

# --- Estrai IP con tentativi falliti da journalctl incrementale ---
journalctl _COMM=sshd --since "$SINCE" | grep "Failed password" | \
awk '{for(i=1;i<=NF;i++) if ($i ~ /^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$/) print $i}' | \
sort | uniq -c | awk -v soglia="$SOGLIA" '$1 >= soglia {print $2}' > /tmp/new_failed_ips.txt

# --- Aggiunta e notifica ---
while read ip; do
  if ! ipset test ssh_blacklist "$ip" &>/dev/null; then
    ipset add ssh_blacklist "$ip"
    echo "$(date '+%F %T') BANNED: $ip" >> "$LOGFILE"
    logger "[SSH-BAN] IP bannato: $ip"

    # Telegram
    MSG="🚫 Nuovo IP bannato per brute force SSH:\n$ip"
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d chat_id="${TELEGRAM_CHAT_ID}" \
      -d text="$MSG" \
      -d parse_mode="Markdown" > /dev/null
  fi
done < /tmp/new_failed_ips.txt

rm -f /tmp/new_failed_ips.txt
```

Rendi eseguibile:

```bash
sudo chmod +x /usr/local/bin/ssh-ban.sh
```

---

## 5️⃣ Automatizza con cron

Modifica il crontab root:

```bash
sudo crontab -e
```

Aggiungi:

```cron
*/10 * * * * /usr/local/bin/ssh-ban.sh
```

---

## 6️⃣ Rendi persistente iptables e ipset

Salva le regole iptables:

```bash
sudo netfilter-persistent save
```

Salva la lista ipset:

```bash
sudo ipset save > /etc/ipset.conf
```

---

## 7️⃣ Ripristino ipset al boot

Crea file systemd:

```bash
sudo vi /etc/systemd/system/ipset-restore.service
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

Mostra IP bannati:

```bash
sudo ipset list ssh_blacklist
```

Controlla log:

```bash
cat /var/log/ssh-ban.log
```

Controlla cron:

```bash
grep ssh-ban /var/log/syslog
```

---

## 📎 File utili

| File                                    | Scopo                      |
| --------------------------------------- | -------------------------- |
| `/usr/local/bin/ssh-ban.sh`             | Script principale          |
| `/var/lib/ssh-ban/last_check.timestamp` | Timestamp ultima scansione |
| `/var/log/ssh-ban.log`                  | Log ban locali             |
| `/etc/ipset.conf`                       | Lista IP persistente       |
| `ipset-restore.service`                 | Restore al boot            |

---

## 🧠 Note finali

* Lo script è **incrementale**, leggero ed efficiente anche su Raspberry Pi
* Il file `last_check.timestamp` previene analisi ripetute
* Funziona anche in abbinata con altri sistemi di logging (fail2ban, firewall)
