# Tests de transfert BLE OTS — RailNet200

## Architecture

```
Ubuntu (PC)  ←—— BLE OTS L2CAP CoC ——→  NINA-B301 (nRF52840)  ←—— UART SOTP 115200 ——→  iMX6 (Yocto Linux)
   ots_transfer.py                        peripheral_ots_railnet               sotp-bridge / sotp-send
```

## Prérequis

### Ubuntu (PC de test)
```bash
cd ~/TRAVAIL/git_soft_dev/SIOT10066-stimio-linux-menus/tools
source .venv312/bin/activate
pip install bleak
```

### iMX6 (cible RailNet200)
- `sotp-bridge` installé et activé (`systemctl start sotp-bridge`)
- `sotp-send` installé (même recette bitbake)
- **IMPORTANT** : le baudrate de sotp-bridge doit matcher la NINA (115200 actuellement)
- Modem Quectel (`ppp`) peut causer des interférences UART → arrêter avec `systemctl stop ppp` si problèmes

### NINA-B301 (firmware Zephyr)
- Firmware : `peripheral_ots_railnet`
- Board : `railnet200_nina_b301`
- Build : `west build -b railnet200_nina_b301 zephyr/samples/bluetooth/peripheral_ots_railnet -p && west flash`

## Test 1 : Send Ubuntu → iMX6 (petit fichier)

### Commandes

```bash
# Ubuntu — envoyer un fichier de 522 bytes
python3 ots_transfer.py DE:CA:F1:5B:AD:11 send --file /etc/hostname --path_dest /tmp/test_small.txt
```

### Vérification

```bash
# iMX6
md5sum /tmp/test_small.txt
# Ubuntu
md5sum /etc/hostname
# Les MD5 doivent matcher
```

### Résultat attendu
- 1 chunk OTS (< 30 KB)
- Débit émission : ~490 kB/s
- Durée totale : ~5s (dont connexion BLE)

## Test 2 : Send Ubuntu → iMX6 (1 MB)

### Commandes

```bash
# Générer un fichier de 1 MB
dd if=/dev/urandom of=/tmp/test_1M.bin bs=1024 count=1024
md5sum /tmp/test_1M.bin

# Envoyer
python3 ots_transfer.py DE:CA:F1:5B:AD:11 send --file /tmp/test_1M.bin --path_dest /tmp/test_1M_received.bin
```

### Vérification

```bash
# iMX6
md5sum /tmp/test_1M_received.bin
wc -c /tmp/test_1M_received.bin
# Doit afficher 1048576 bytes et le même MD5
```

### Résultat attendu
- 35 chunks de ~30 KB chacun
- Débit émission BLE : ~440-490 kB/s
- Durée totale : ~120s (overhead = connexion BLE + OACP Create/Write par chunk)

## Test 3 : Receive iMX6 → Ubuntu (petit fichier)

### Commandes

```bash
# iMX6 — envoyer un objet dans le pool OTS de la NINA
systemctl stop sotp-bridge    # libérer l'UART
sotp-send /etc/stimio/config.json
systemctl start sotp-bridge

# Ubuntu — recevoir le premier objet OTS
python3 ots_transfer.py DE:CA:F1:5B:AD:11 receive -o /tmp/config_received.json
```

### Vérification

```bash
# Ubuntu
md5sum /tmp/config_received.json
# iMX6
md5sum /etc/stimio/config.json
# Les MD5 doivent matcher
```

### Résultat attendu
- Object Name : `config.json`
- Object Size : 522 bytes
- MD5 : `e13fad91637b1e88235325f045b1a705` (pour ce config.json spécifique)
- Durée : ~5s

### Résultat validé (2026-04-08)
```
=== Résultat: 3/3 PASS, 0/3 FAIL ===
```

## Test 4 : Receive iMX6 → Ubuntu (fichier plus grand)

### Commandes

```bash
# iMX6 — préparer un fichier de test
dd if=/dev/urandom of=/tmp/test_send.bin bs=1024 count=30
md5sum /tmp/test_send.bin
systemctl stop sotp-bridge
sotp-send /tmp/test_send.bin
systemctl start sotp-bridge

# Ubuntu
python3 ots_transfer.py DE:CA:F1:5B:AD:11 receive -o /tmp/test_received.bin
md5sum /tmp/test_received.bin
```

### Limite
- `sotp-send` est limité à 32 KB (taille max d'un objet OTS dans le pool NINA)
- Préférer la commande `request` qui gère automatiquement les gros fichiers

## Test 5 : Request fichier spécifique (commande unifiée)

### Commandes

```bash
# Ubuntu — demander un fichier par son chemin
python3 ots_transfer.py DE:CA:F1:5B:AD:11 request --path /etc/stimio/config.json -o /tmp/config.json
```

### Résultat attendu
- Le script affiche le MD5 directement
- 1 chunk pour les fichiers < 3.8 KB
- Durée : ~35s

### Résultat validé (2026-04-08)
```
MD5 : e13fad91637b1e88235325f045b1a705 — PASS (3/3 runs)
```

## Test 6 : Request fichier 100 KB (multi-chunk)

### Commandes

```bash
# iMX6 — créer un fichier de test + arrêter le modem USB
systemctl stop ppp
dd if=/dev/urandom of=/tmp/test_100k.bin bs=1024 count=100
md5sum /tmp/test_100k.bin

# Ubuntu
python3 ots_transfer.py DE:CA:F1:5B:AD:11 request --path /tmp/test_100k.bin -o /tmp/test_100k_recv.bin
```

### Résultat attendu
- 27 chunks de ~3.8 KB chacun
- MD5 match
- Durée : ~15 min (limité par les crédits L2CAP BlueZ, voir optimisations)

### Résultat validé (2026-04-09)
```
Taille : 102400 bytes, 27 chunks
MD5    : 0d55302f9f6354d0fffba5fa1b8e12c5 — PASS
Durée  : 896s (~15 min)
```

## Test 7 : Request fichier inexistant (erreur)

### Commandes

```bash
python3 ots_transfer.py DE:CA:F1:5B:AD:11 request --path /tmp/does_not_exist.bin -o /tmp/nope.bin
```

### Résultat attendu
- Timeout 30s puis message d'erreur

### Résultat validé (2026-04-08)
```
ERREUR : aucun objet reçu après 15.4s (fichier introuvable ou erreur iMX6)
```

## Bugs résolus pendant les tests

### 1. UART RX polling perd des octets (CRC mismatch systématique)
- **Symptôme** : `sotp_rx: CRC mismatch` sur chaque trame reçue par la NINA
- **Cause** : `uart_poll_in()` ne bufferise qu'un octet. À 115200 baud (1 octet / 87µs), le `k_sleep(K_MSEC(1))` perd ~11 octets par cycle
- **Fix** : Passage en mode interrupt-driven (`uart_irq_callback_set()`) avec ring buffer dans `uart_relay.c`

### 2. Pins RTS/CTS dans pinctrl bloquent la réception UART
- **Symptôme** : la NINA ne reçoit aucun octet (aucun log, même pas de CRC mismatch)
- **Cause** : les pins `UART_RTS`/`UART_CTS` déclarés dans le pinctrl sont programmés dans le registre PSEL du UARTE nRF52840, même sans `hw-flow-control`. Le matériel bloque la réception en attendant CTS
- **Fix** : retirer `NRF_PSEL(UART_RTS, ...)` et `NRF_PSEL(UART_CTS, ...)` du pinctrl quand `hw-flow-control` est désactivé

### 3. OTS Object créé par le serveur a size.cur = 0
- **Symptôme** : OLCP GoTo Next retourne Success mais Object Name/Size retourne toujours "Directory" (l'objet ajouté via `sotp-send` est invisible)
- **Cause** : `obj_created()` retournait `size.cur = 0` pour les objets créés par `ots_handler_add_object_from_uart()`. Zephyr OTS considère l'objet comme vide
- **Fix** : ajout de `pending_data_len` dans `ots_handler.c` ; `obj_created()` retourne `size.cur = pending_data_len` quand `pending_name` est défini

### 4. L2CAP CoC SDU_LEN header non strippé en réception
- **Symptôme** : fichier reçu contient des octets parasites `\x00\x01` au début et au milieu
- **Cause** : Zephyr OTS envoie les données via `bt_gatt_ots_l2cap_send()` qui ajoute un SDU_LEN header (2 bytes LE16) en tête de chaque SDU. BlueZ `SOCK_SEQPACKET` ne strip pas ce header
- **Fix** : parser les SDU_LEN headers dans le flux reçu dans `ots_transfer.py` (`struct.unpack("<H", raw[:2])`)

### 5. Pool OTS vidé à la déconnexion BLE
- **Symptôme** : après un receive réussi, le prochain receive ne trouve que le Directory (l'objet a disparu)
- **Cause** : `ots_handler_cleanup_on_disconnect()` supprime tous les objets du pool. L'objet envoyé par `sotp-send` est perdu
- **Contournement** : relancer `sotp-send` avant chaque `receive`
- **Amélioration future** : ne pas supprimer les objets ajoutés par l'iMX6 lors du cleanup

### 6. sotp-bridge à 1 Mbaud vs NINA à 115200
- **Symptôme** : sotp-bridge ne reçoit pas les trames SOTP OBJ_FROM_BLE de la NINA
- **Cause** : sotp-bridge.c compilé avec `B1000000` alors que la NINA est à 115200
- **Fix** : recompiler sotp-bridge avec `B115200` (nécessite rebuild Yocto)

### 7. ACK confusion dans multi-chunk FILE_REQUEST
- **Symptôme** : CRC mismatch reproductible `recv=0xaaf9 calc=0xc2f9` en attente d'ACK
- **Cause** : l'ACK normal (0x03) de `add_object` arrivait dans le buffer UART avant le DELETE_ACK (0x07), corrompant la state machine SOTP
- **Fix** : flush du buffer UART (50ms) + reset state machine dans `wait_for_ack()` avant d'attendre le DELETE_ACK

### 8. bt_ots_obj_delete passe conn=NULL au callback
- **Symptôme** : DELETE_ACK jamais envoyé, `if (conn)` toujours faux
- **Cause** : Zephyr `bt_ots_obj_delete()` passe toujours NULL pour conn
- **Fix** : utiliser un flag `cleaning_up` au lieu de tester conn

### 9. BlueZ SOCK_SEQPACKET limite les crédits L2CAP à ~16
- **Symptôme** : objets OTS de 30 KB reçus partiellement (~6.7 KB seulement)
- **Cause** : BlueZ n'accorde que ~16 crédits L2CAP initiaux au récepteur
- **Contournement** : réduire les chunks à 3800 bytes (15 SDUs, dans les 16 crédits)

## Commandes utiles pour le debug

### Vérifier que la NINA reçoit sur l'UART
```bash
# iMX6 — envoyer une trame SOTP STATUS minimale
stty -F /dev/ttymxc7 115200 raw -echo
echo -ne '\xaa\x55\x05\x00\x00\x00\x00\xdc\x16' > /dev/ttymxc7
# RTT NINA doit afficher : sotp_dispatch: STATUS from iMX6 (len=0)
```

### Voir les logs NINA en temps réel (via J-Link RTT)
```bash
JLinkRTTClient
# ou
JLinkRTTLogger -Device NRF52840_XXAA -If SWD -Speed 4000
```

### Vérifier le baudrate du sotp-bridge
```bash
# iMX6
journalctl -u sotp-bridge --no-pager -n 5
# Doit afficher : opened /dev/ttymxc7 at 115200 baud
```

### Arrêter le modem USB (réduit les interférences UART)
```bash
# iMX6
systemctl stop ppp
```
