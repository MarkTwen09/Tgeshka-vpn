#!/bin/bash
# add_client.sh - добавляет нового клиента в Tgeshka VPN

set -e

NEW_CLIENT=$1
if [ -z "$NEW_CLIENT" ]; then
    echo "Использование: $0 <имя_клиента>"
    exit 1
fi

# Генерация ключей клиента
cd /etc/amneziawg
wg genkey | tee ${NEW_CLIENT}_private.key | wg pubkey > ${NEW_CLIENT}_public.key
CLIENT_PRIV=$(cat ${NEW_CLIENT}_private.key)
CLIENT_PUB=$(cat ${NEW_CLIENT}_public.key)

# Определение следующего доступного IP
LAST_IP=$(grep -E "AllowedIPs = [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" /etc/amneziawg/wg0.conf | tail -1 | grep -oP "\d+\.\d+\.\d+\.\d+" | cut -d. -f4)
if [ -z "$LAST_IP" ]; then
    NEXT_IP=2
else
    NEXT_IP=$((LAST_IP+1))
fi
CLIENT_IP="10.0.0.$NEXT_IP"

# Добавление пира в конфиг сервера
cat >> /etc/amneziawg/wg0.conf <<EOF

[Peer]
PublicKey = $CLIENT_PUB
AllowedIPs = $CLIENT_IP/32
EOF

# Перезапуск интерфейса
awg-quick down wg0
awg-quick up wg0

# Создание конфигурации клиента
PUBLIC_IP=$(curl -s ifconfig.me)
SERVER_PUB=$(cat server_public.key)

mkdir -p /root/tgeshka_configs
CLIENT_CONF="/root/tgeshka_configs/$NEW_CLIENT.conf"
cat > $CLIENT_CONF <<EOF
[Interface]
PrivateKey = $CLIENT_PRIV
Address = $CLIENT_IP/32
DNS = 1.1.1.1, 9.9.9.9

[Peer]
PublicKey = $SERVER_PUB
Endpoint = $PUBLIC_IP:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
EOF

echo "Клиент $NEW_CLIENT добавлен. Конфиг: $CLIENT_CONF"
qrencode -t ansiutf8 < $CLIENT_CONF
