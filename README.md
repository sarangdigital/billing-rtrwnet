# billing-rtrwnet
Web Billing RTRW-Net


# L2TP Ubuntu Server – Auto Reconnect (Stable & Production)

Dokumentasi ini menjelaskan setup **L2TP Client di Ubuntu Server** dengan konfigurasi lengkap,
stabil, dan siap produksi.

---

## 1. Informasi Koneksi

| Item | Nilai |
|-----|------|
| L2TP Server | 10.70.90.1 |
| PPP Username | sarangid |
| PPP Password | sarangidubuntuserver |
| WAN Interface | eno1 |
| WAN Gateway | 10.1.68.1 |
| L2TP Name | l2tp-public |

---

## 2. Install Paket

```bash
apt update
apt install -y xl2tpd ppp systemd-resolved
```

---

## 3. Konfigurasi PPP (Username & Password)

### File
```bash
/etc/ppp/chap-secrets
```

### Isi
```text
sarangid * sarangidubuntuserver *
```

### Permission
```bash
chmod 600 /etc/ppp/chap-secrets
```

---

## 4. Konfigurasi PPP Options

### File
```bash
/etc/ppp/options.l2tpd.client
```

### Isi
```text
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-chap
noccp
noauth
mtu 1400
mru 1400
usepeerdns
persist
```

---

## 5. Konfigurasi xl2tpd

### File
```bash
/etc/xl2tpd/xl2tpd.conf
```

### Isi
```ini
[global]
port = 1701

[lac l2tp-public]
lns = 10.70.90.1
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
```

---

## 6. Script Final (Auto Fix & Auto Reconnect)

### Lokasi
```bash
/usr/local/bin/l2tp-autofix.sh
```

### Isi Script
```bash
#!/bin/bash
set -e

L2TP_NAME="l2tp-public"
L2TP_SERVER="10.70.90.1"
WAN_IF="eno1"
WAN_GW="10.1.68.1"
CRON_FILE="/etc/cron.d/l2tp-autoreconnect"

ip route replace ${L2TP_SERVER} via ${WAN_GW} dev ${WAN_IF}

cat <<'EOF' > /etc/ppp/ip-up.d/99-l2tp-route
#!/bin/sh
ip route replace default dev "$PPP_IFACE"
resolvectl dns "$PPP_IFACE" 8.8.8.8 1.1.1.1
resolvectl default-route "$PPP_IFACE" yes
EOF
chmod +x /etc/ppp/ip-up.d/99-l2tp-route

systemctl restart xl2tpd
sleep 2
echo "c ${L2TP_NAME}" > /var/run/xl2tpd/l2tp-control

cat <<EOF > ${CRON_FILE}
* * * * * root ip -o link show | grep -q 'ppp[0-9]' || echo "c ${L2TP_NAME}" > /var/run/xl2tpd/l2tp-control
EOF
chmod 644 ${CRON_FILE

echo "DONE"
```

Aktifkan:
```bash
chmod +x /usr/local/bin/l2tp-autofix.sh
/usr/local/bin/l2tp-autofix.sh
```

---

## 7. Verifikasi

```bash
ip a | grep ppp
ip route
curl ifconfig.me
```

---

## 8. Auto Reconnect Logic

- PPP terputus → cron mendeteksi tidak ada pppX
- xl2tpd otomatis dial ulang
- Aman walau ppp0 / ppp1 berubah
- Aman reboot

---

## 9. Catatan

- Jangan hapus route ke server L2TP
- Jangan gunakan cron berbasis ping
- Satu cron cukup (jangan bikin xl2tpd stres 😄)

---

## 10. Status Akhir

✔ Stabil  
✔ Production ready  
✔ Anti ppp0 / ppp1 drama  
✔ Sekali set, tinggal ngopi ☕
