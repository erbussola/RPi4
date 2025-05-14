# Guida completa: Swap su iSCSI con watchdog e systemd su Raspberry Pi 4

## Premessa

Questa guida descrive la configurazione completa di uno **swap file su LUN iSCSI** per un **Raspberry Pi 4**, evitando l'usura della microSD. Tutta la configurazione utilizza `systemd` invece di `/etc/fstab` per maggiore robustezza, ed è dotata di un watchdog con invio email in caso di errore.

---

## 1. Requisiti

* Raspberry Pi 4 con Raspbian o Debian-like OS
* LUN iSCSI disponibile e accessibile (es. su NAS QNAP)
* Connessione di rete stabile
* Pacchetti installati:

  ```bash
  sudo apt install open-iscsi mailutils
  ```

---

## 2. Login automatico alla LUN iSCSI

### 2.1 Scansione e login iniziale

```bash
sudo iscsiadm -m discovery -t sendtargets -p <IP_DEL_TARGET>
sudo iscsiadm -m node -T <NOME_TARGET> -p <IP_DEL_TARGET> --login
```

### 2.2 Verifica UUID del dispositivo iSCSI

```bash
lsblk -f
# Oppure:
sudo blkid
```

Esempio di UUID: `84a2b5ba-dc07-4a8e-9d48-b2e0769fe1f5`

---

## 3. Configurazione swap sulla LUN iSCSI

### 3.1 Creazione e attivazione iniziale

```bash
sudo mkswap /dev/disk/by-uuid/84a2b5ba-dc07-4a8e-9d48-b2e0769fe1f5
sudo swapon /dev/disk/by-uuid/84a2b5ba-dc07-4a8e-9d48-b2e0769fe1f5
```

### ⚠️ Nota importante

Non configurare `/etc/fstab`, per evitare problemi di boot: l'attivazione avverrà tramite `systemd`.

---

## 4. Unit personalizzata: iscsi-login.service

Percorso: `/etc/systemd/system/iscsi-login.service`

```ini
[Unit]
Description=Login iSCSI al boot
DefaultDependencies=no
After=network-online.target
Before=swap-iscsi.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/iscsiadm -m node --loginall=automatic
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

Attiva con:

```bash
sudo systemctl enable iscsi-login.service
```

---

## 5. Unit per swap iSCSI

Percorso: `/etc/systemd/system/swap-iscsi.service`

```ini
[Unit]
Description=Attivazione dello swap su iSCSI
Requires=iscsi-login.service
After=iscsi-login.service

[Service]
Type=oneshot
ExecStart=/sbin/swapon /dev/disk/by-uuid/84a2b5ba-dc07-4a8e-9d48-b2e0769fe1f5
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

Abilita:

```bash
sudo systemctl enable swap-iscsi.service
```

---

## 6. Watchdog per monitoraggio swap iSCSI

### Script: `/usr/local/bin/swap-watchdog.sh`

```bash
#!/bin/bash
SWAP_DEVICE="/dev/disk/by-uuid/84a2b5ba-dc07-4a8e-9d48-b2e0769fe1f5"
MAX_RETRIES=3
EMAIL_TO="f.biscossi@gmail.com"
LOGFILE="/var/log/swap-watchdog.log"
HOSTNAME=$(hostname)

send_email() {
    local SUBJECT="$1"
    local BODY="$2"
    echo "$(date) - Sending email: $SUBJECT" >> /tmp/send_email.log
    echo "$BODY" | sudo -u pi /usr/bin/mail -s "$SUBJECT" "$EMAIL_TO"
    if [ $? -eq 0 ]; then
        echo "$(date) - Email sent successfully." >> /tmp/send_email.log
    else
        echo "$(date) - Email send failed." >> /tmp/send_email.log
    fi
}

while true; do
    if ! swapon --show | grep -q "$SWAP_DEVICE"; then
        echo "$(date) - Swap offline, tentativo di riattivazione..." | tee -a "$LOGFILE"

        for ((i=1; i<=MAX_RETRIES; i++)); do
            swapoff -a
            if swapon "$SWAP_DEVICE"; then
                MSG="Swap riattivato con successo"
                echo "$(date) - $MSG" | tee -a "$LOGFILE"
                send_email "[SUCCESS] Swap iSCSI riattivato su $HOSTNAME" "$MSG"
                break
            else
                echo "$(date) - Tentativo $i fallito" | tee -a "$LOGFILE"
                systemctl restart iscsi-login.service
                sleep 5
            fi
        done

        if ! swapon --show | grep -q "$SWAP_DEVICE"; then
            MSG="ERRORE CRITICO: Impossibile riattivare lo swap iSCSI su $HOSTNAME"
            echo "$(date) - $MSG" | tee -a "$LOGFILE"
            send_email "[CRITICAL] Swap iSCSI non ripristinato su $HOSTNAME" "$MSG"
        fi
    fi
    sleep 30
done
```

Rendi eseguibile:

```bash
sudo chmod +x /usr/local/bin/swap-watchdog.sh
```

---

### Unit: `/etc/systemd/system/swap-watchdog.service`

```ini
[Unit]
Description=Swap iSCSI Watchdog
Wants=swap-iscsi.service
After=swap-iscsi.service
OnFailure=check-iscsi-swap.service

[Service]
Type=simple
ExecStart=/usr/local/bin/swap-watchdog.sh
Restart=always
RestartSec=60s

[Install]
WantedBy=multi-user.target
```

Attiva con:

```bash
sudo systemctl enable --now swap-watchdog.service
```

---

## 7. Verifica stato

```bash
swapon --show
systemctl status iscsi-login.service
systemctl status swap-iscsi.service
systemctl status swap-watchdog.service
```

---

## 8. Backup consigliati

* `/etc/systemd/system/*.service`
* `/usr/local/bin/swap-watchdog.sh`
* UUID dispositivo swap

---

## 9. Possibili estensioni

* Creazione di un timer settimanale con `systemd.timer` per check swap + email
* Dashboard Grafana/Prometheus per monitoraggio più evoluto

---

## 10. Troubleshooting

* Controlla `/var/log/swap-watchdog.log` e `/tmp/send_email.log`
* Usa `journalctl -u nome-servizio` per logs dettagliati

---

## Autore

Fabrizio Biscossi – Technical Team Leader & Infrastructure Specialist – Linux & Virtualization

> Ultimo aggiornamento: Maggio 2025
