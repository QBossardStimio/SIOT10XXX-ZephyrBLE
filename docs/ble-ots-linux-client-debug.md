# BLE OTS Linux Client — Debugging Session

## Contexte

Test du pipeline complet de transfert de fichier via BLE OTS :

```
PC Ubuntu (ots_send.py) → BLE OTS → NINA-B301 (Zephyr) → UART SOTP → iMX6 RailNet200
```

Outil côté PC : `tools/ots_send.py` (Python + Bleak)
Firmware côté NINA : `zephyr/samples/bluetooth/peripheral_ots_railnet/`
Adresse BLE NINA : `DE:CA:F1:5B:AD:11` (Random Static)
Adaptateur PC : `hci0` adresse publique `A0:B3:39:B4:00:95`
OS PC : Ubuntu, BlueZ 5.72, kernel 6.17

---

## Problèmes résolus

### 1. `CONFIG_BT_L2CAP_DYNAMIC_CHANNEL` manquant dans `prj.conf`

**Symptôme :** Timeout L2CAP CoC côté PC, aucun log L2CAP côté NINA.

**Cause :** Sans `CONFIG_BT_L2CAP_DYNAMIC_CHANNEL=y`, Zephyr n'enregistre pas le serveur PSM 0x0025. Le PSM n'existe pas sur la NINA → toute tentative de connexion L2CAP CoC est refusée au niveau BLE.

**Correction :**
```
# prj.conf
CONFIG_BT_L2CAP_DYNAMIC_CHANNEL=y
```

**Fichier :** `zephyr/samples/bluetooth/peripheral_ots_railnet/prj.conf`

---

### 2. Advertising non relancé après déconnexion BLE

**Symptôme :** Après une déconnexion, la NINA n'advertise plus. Impossible de se reconnecter sans reboot.

**Cause :** Le callback `disconnected()` dans `main.c` ne relançait pas `bt_le_adv_start()`.

**Correction :**
```c
static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    // ... log ...
    atomic_dec(&stat_ble_connections);

    int err = bt_le_adv_start(BT_LE_ADV_CONN_FAST_1,
                  ad, ARRAY_SIZE(ad),
                  sd, ARRAY_SIZE(sd));
    if (err) {
        LOG_ERR("bt_le_adv_start after disconnect failed: %d", err);
    } else {
        LOG_INF("Advertising restarted");
    }
}
```

**Fichier :** `zephyr/samples/bluetooth/peripheral_ots_railnet/src/main.c`

---

### 3. `west build --pristine=always` échoue : `BOARD is not being defined`

**Symptôme :**
```
CMake Error: BOARD is not being defined on the CMake command-line
```

**Cause :** `--pristine=always` efface le cache CMake, y compris le board précédemment configuré.

**Correction :** Toujours spécifier `-b` explicitement avec `--pristine` :
```bash
west build -d build_ots --pristine=always -b railnet200_nina_b301/nrf52840 \
    zephyr/samples/bluetooth/peripheral_ots_railnet
```

---

### 4. BlueZ systemd override — mauvais chemin `bluetoothd`

**Symptôme :** `bluetooth.service` démarre puis s'arrête immédiatement avec `status=203/EXEC`.

**Cause :** Le chemin utilisé était `/usr/sbin/bluetooth/bluetoothd` (un répertoire en trop).

**Correction :**
```ini
# /etc/systemd/system/bluetooth.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/sbin/bluetoothd --experimental
```

**Commandes :**
```bash
sudo systemctl daemon-reload
sudo systemctl restart bluetooth
```

---

### 5. `sock.bind()` — format tuple invalide pour `BTPROTO_L2CAP`

**Symptôme :** `OSError: bind(): wrong format` avec un tuple à 4 éléments.

**Cause :** Python `socket` pour `AF_BLUETOOTH / BTPROTO_L2CAP` n'accepte que des tuples à 2 éléments `(addr, psm)` pour `bind()`, pas 4 éléments.

**Correction :**
```python
sock.bind(("00:00:00:00:00:00", 0))  # 2 éléments uniquement
```

---

### 6. `sock.connect()` — format tuple invalide pour bdaddr_type LE

**Symptôme :** `OSError: connect(): wrong format` avec un tuple à 4 éléments `(addr, psm, cid, bdaddr_type)`.

**Cause :** Python `socket` ne supporte pas le 4-tuple pour `BTPROTO_L2CAP` connect. Il faut construire la struct `sockaddr_l2` manuellement via `ctypes.libc.connect()`.

**Correction :**
```python
import ctypes, ctypes.util, struct

libc = ctypes.CDLL(ctypes.util.find_library("c"), use_errno=True)

def l2cap_connect_raw(fd, addr_str, psm, bdaddr_type, timeout=10.0):
    parts = [int(x, 16) for x in addr_str.split(":")]
    bdaddr = bytes(reversed(parts))
    # struct sockaddr_l2 { sa_family(H) psm(H) bdaddr(6s) cid(H) bdaddr_type(B) }
    sockaddr = struct.pack("<HH6sHB", 31, psm, bdaddr, 0, bdaddr_type)
    sockaddr_buf = ctypes.create_string_buffer(sockaddr)
    ret = libc.connect(fd, sockaddr_buf, len(sockaddr))
    # ... gestion EINPROGRESS ...
```

---

### 7. `EINPROGRESS (115)` non géré sur `libc.connect()`

**Symptôme :** `BlockingIOError: [Errno 115] Operation now in progress`

**Cause :** Le socket L2CAP est non-bloquant par défaut. `EINPROGRESS` signifie que la connexion est en cours — ce n'est pas une erreur.

**Correction :** Attendre via `select()` + vérifier `SO_ERROR` :
```python
import select
if errno != 115:  # EINPROGRESS
    raise OSError(errno, ...)
ready = select.select([], [fd], [fd], timeout)
if not ready[1] and not ready[2]:
    raise TimeoutError("L2CAP CoC connect timeout")
err = sock.getsockopt(socket.SOL_SOCKET, socket.SO_ERROR)
if err != 0:
    raise OSError(err, ...)
```

---

### 8. `bt_le_adv_start after disconnect failed: -12` (ENOMEM)

**Symptôme :** Après déconnexion, `bt_le_adv_start()` échoue avec `-12`.

**Cause :** L'objet OTS créé via OACP Create reste dans le pool (`in_use=true`) après la déconnexion. Le pool est plein → ENOMEM sur une opération interne Zephyr liée à l'advertising.

**État :** Partiellement contourné. `ots_conn_disconnected()` est appelé par Zephyr OTS à la déconnexion (visible dans les logs DBG), ce qui devrait nettoyer le contexte L2CAP. Mais l'objet dans `obj_pool[]` reste `in_use=true` car `obj_deleted()` n'est appelé que sur OACP Delete explicite.

**Solution :** Implémenté — `ots_handler_cleanup_on_disconnect()` appelée depuis `main.c::disconnected()`. Remet tous les slots `in_use=false`, appelle `bt_ots_obj_delete()` pour chaque slot actif, et remet `obj_count=0`.

**Fichier :** `zephyr/samples/bluetooth/peripheral_ots_railnet/src/ots_handler.c`

---

### 9. SDU_LEN header manquant — BlueZ SOCK_SEQPACKET ne le préfixe pas

**Symptôme :** Chunks 2+ échouent côté NINA avec `L2CAP RX PDU total exceeds SDU` (`sdu_len=1`, `sdu_len=2`...).

**Cause :** BlueZ `SOCK_SEQPACKET` L2CAP CoC **n'ajoute PAS** automatiquement le SDU_LEN header (2 bytes LE16 au début du premier PDU du SDU). Zephyr `l2cap_chan_le_recv_seg_direct()` fait `pull_le16(seg)` sur les 2 premiers bytes pour lire `_sdu_len`. Sans le header, Zephyr lit les bytes de données comme `_sdu_len` → valeur absurde → rejet.

**Diagnostic :** Log ajouté dans `l2cap.c:2688` a montré `sdu_len=1` pour chunk 2 (= `chunk_idx` encodé en LE16, dont le premier byte vaut `0x01`).

**Correction :**
```python
# Dans send_l2cap() — ots_transfer.py
pdu = struct.pack("<H", len(payload)) + payload
sock.send(pdu)
```

**Fichier :** `tools/ots_transfer.py`

---

### 10. `bt_ots_obj_delete` retourne -EBUSY depuis `obj_write` callback

**Symptôme :** Pool OTS saturé après 4 chunks (pool 4/4), OACP Create échoue avec `Insufficient Resources`.

**Cause :** `bt_ots_obj_delete()` vérifie `obj->state.type == BT_GATT_OTS_OBJECT_IDLE_STATE`. Or, `oacp_write_common()` (ots_oacp.c:601) remet l'objet en IDLE **après** le retour du callback `obj_write`. Appel direct depuis `obj_write` → objet encore en WRITE state → -EBUSY.

**Correction :** Différer la suppression via `k_work_submit()` :
```c
static uint64_t pending_delete_id;
static K_WORK_DEFINE(delete_work, delete_work_handler);

static void delete_work_handler(struct k_work *work) {
    bt_ots_obj_delete(ots_instance, pending_delete_id);
}

// Dans obj_write, quand remaining == 0 :
pending_delete_id = id;
k_work_submit(&delete_work);
```

Le work handler s'exécute sur la system workqueue après que la stack OTS a remis l'objet en IDLE.

**Fichier :** `zephyr/samples/bluetooth/peripheral_ots_railnet/src/ots_handler.c`

---

### 11. `rename()` EXDEV dans sotp-bridge — cross-device link

**Symptôme :** `rename /tmp/sotp_XXXXXX -> /var/lib/sotp-bridge/ble_obj_000000.bin failed: Invalid cross-device link`

**Cause :** `mkstemp()` crée le fichier temporaire dans `/tmp` (tmpfs). La destination `/var/lib/sotp-bridge` est sur le rootfs. `rename()` ne fonctionne pas entre filesystems différents → `EXDEV`.

**Correction :** Fallback copy+unlink quand `errno == EXDEV` :
```c
if (rename(src, dst) == 0) { /* OK */ }
else if (errno == EXDEV) {
    // open(src), open(dst, O_CREAT|O_TRUNC), read/write loop, close, unlink(src)
}
```

**Fichier :** `layers/STIMIODEV/SIOT10069-stimio-meta-app/recipes-connectivity/sotp-bridge/files/sotp-bridge.c`

---

### 12. HCI ACL buffer overflow avec SDU multi-PDU (> 245 bytes)

**Symptôme :** `Not enough buffer space for L2CAP data: cont_len=251 tailroom=4` et `Unexpected L2CAP continuation` côté NINA.

**Cause :** Un `sock.send()` de plus de 245 bytes (MPS NINA) provoque une fragmentation BlueZ en plusieurs PDUs. Le contrôleur nRF52840 n'a pas assez de buffers ACL pour reassembler les fragments HCI.

**Correction :** Limiter chaque `sock.send()` à 243 bytes de données (+ 2 SDU_LEN = 245 = MPS). Un send = un SDU = un PDU L2CAP.
```python
L2CAP_DATA_PER_SDU = 241  # 245 MPS - 2 SDU_LEN - 2 L2CAP header overhead
```

**Fichier :** `tools/ots_transfer.py`

---

### 13. `oacp_write_seg_cb` : mauvais offset avec multi-SDU (SEG_RECV)

**Symptôme :** Données corrompues dans l'objet OTS — chaque SDU écrase l'offset 0 au lieu d'écrire séquentiellement.

**Cause :** `oacp_write_seg_cb()` dans `ots_oacp.c` utilisait `seg_offset` qui repart à 0 pour chaque nouveau SDU. Avec `CONFIG_BT_L2CAP_SEG_RECV`, chaque SDU est un segment indépendant.

**Correction :** Utiliser `write_op->recv_len` (octets accumulés depuis le début de l'OACP Write) au lieu de `seg_offset` :
```c
write_op = &ots->cur_obj->state.write_op;
offset = write_op->oacp_params.offset + write_op->recv_len;
```

**Fichier :** `zephyr/subsys/bluetooth/services/ots/ots_oacp.c`

---

### 14. Pool OTS saturé — k_work delete non exécuté avant le chunk suivant

**Symptôme :** `Insufficient Resources` sur OACP Create du chunk 4, alors que le pool fait 4 slots.

**Cause :** Le `k_work_submit(&delete_work)` du chunk N ne s'exécute pas avant que le client envoie OACP Create du chunk N+1 (temps entre les chunks < scheduling du k_work).

**Correction :** Ajouter un `asyncio.sleep(0.05)` entre les chunks côté Python pour laisser le temps au k_work de supprimer l'objet.

**Fichier :** `tools/ots_transfer.py`

---

### 15. Pins RTS/CTS dans pinctrl bloquent la réception UART

**Symptôme :** La NINA ne reçoit **aucun octet** sur l'UART. Aucun log CRC mismatch, rien — silence total.

**Cause :** Les pins `UART_RTS` (P0.31) et `UART_CTS` (P1.12) étaient déclarés dans le `pinctrl` DTS même quand `hw-flow-control` était désactivé. Le driver UARTE nRF52840 programme le registre `PSEL.CTS` dans tous les cas, et le matériel **bloque la réception** en attendant que le signal CTS soit asserté — ce qui ne se produit jamais puisque l'iMX6 ne route pas RTS vers la NINA.

**Diagnostic :**
```bash
# iMX6 — envoyer un octet de test
stty -F /dev/ttymxc7 115200 raw -echo
echo -ne '\xAA' > /dev/ttymxc7
# Rien dans les logs RTT NINA → problème physique/matériel
```

**Correction :** Retirer (commenter) les pins RTS/CTS du pinctrl quand `hw-flow-control` est désactivé :
```dts
uart0_default: uart0_default {
    group1 {
        psels = <NRF_PSEL(UART_RX,  0, 29)>,
                <NRF_PSEL(UART_TX,  1, 13)>;
        /* RTS/CTS omis — décommenter quand hw-flow-control activé */
        /* <NRF_PSEL(UART_RTS, 0, 31)>,
           <NRF_PSEL(UART_CTS, 1, 12)>; */
    };
};
```

**Fichier :** `zephyr/boards/u-blox/railnet200_nina_b301/railnet200_nina_b301/railnet200_nina_b301_nrf52840-pinctrl.dtsi`

---

### 16. UART RX polling perd des octets (CRC mismatch systématique)

**Symptôme :** `sotp_rx: CRC mismatch` sur chaque trame reçue par la NINA depuis `sotp-send`. Les CRC reçus contiennent des valeurs ASCII (`0x203a` = ` :`, `0x6e61` = `na`) — preuve que le payload est décalé.

**Cause :** `uart_poll_in()` ne bufferise **qu'un seul octet** (`EVENTS_RXDRDY`). À 115200 baud, un octet arrive toutes les ~87µs. Le `k_sleep(K_MSEC(1))` entre chaque poll introduit 1ms de latence, pendant laquelle ~11 octets arrivent et sont perdus (écrasés par le suivant).

**Diagnostic :** Les valeurs CRC reçues sont des fragments de texte ASCII du fichier JSON envoyé → le payload est corrompu par des octets manquants → la state machine SOTP perd la synchronisation.

**Correction :** Passer au mode **interrupt-driven** avec un ring buffer de 1024 octets :
```c
static void uart_irq_rx_handler(const struct device *dev, void *user_data)
{
    uint8_t tmp[32];
    int len;
    while ((len = uart_fifo_read(dev, tmp, sizeof(tmp))) > 0) {
        for (int i = 0; i < len; i++) {
            uint32_t next = (rx_ring_head + 1) % SOTP_RX_RING_SIZE;
            if (next == rx_ring_tail) continue;  // ring plein
            rx_ring_buf[rx_ring_head] = tmp[i];
            rx_ring_head = next;
        }
    }
}

// Init:
uart_irq_callback_set(uart_dev, uart_irq_rx_handler);
uart_irq_rx_enable(uart_dev);
```

Le thread `sotp_rx` consomme le ring buffer et alimente la state machine SOTP.

**Fichier :** `zephyr/samples/bluetooth/peripheral_ots_railnet/src/uart_relay.c`

---

### 17. OTS Object créé par le serveur a size.cur = 0 (invisible via OLCP)

**Symptôme :** Après `sotp-send` sur l'iMX6, OLCP GoTo Next retourne Success mais Object Name/Size retourne toujours "Directory". L'objet ajouté par `ots_handler_add_object_from_uart()` est invisible pour le client BLE.

**Cause :** `obj_created()` retournait `created_desc->size.cur = 0` pour **tous** les objets. Les objets créés côté serveur (via `bt_ots_obj_add()`) ont leurs données copiées **après** la création. Mais Zephyr OTS voit `size.cur=0` et considère l'objet comme vide — OLCP le saute ou OACP Read lit 0 bytes.

**Correction :** Ajout d'une variable `pending_data_len` positionnée avant `bt_ots_obj_add()`. Le callback `obj_created()` retourne `size.cur = pending_data_len` quand `pending_name` est défini (création serveur), et `size.cur = 0` sinon (OACP Create client BLE) :
```c
static uint32_t pending_data_len;

// Dans obj_created() :
created_desc->size.cur = pending_name ? pending_data_len : 0;

// Dans ots_handler_add_object_from_uart() :
pending_data_len = (uint32_t)len;
err = bt_ots_obj_add(ots_instance, &param);
pending_data_len = 0;
```

**Fichier :** `zephyr/samples/bluetooth/peripheral_ots_railnet/src/ots_handler.c`

---

### 18. L2CAP CoC SDU_LEN header non strippé en réception

**Symptôme :** Le fichier reçu via `ots_transfer.py receive` contient des octets parasites `\x00\x01` au début et au milieu du contenu. Le MD5 ne matche pas.

**Cause :** Zephyr OTS envoie les données via `bt_gatt_ots_l2cap_send()` qui ajoute un **SDU_LEN header** (2 bytes LE16) en tête de chaque SDU. Pour un objet de 522 bytes, Zephyr envoie 2 SDUs : le premier de 256 bytes data (+ 2 SDU_LEN = 258), le second de 268 bytes data (+ 2 SDU_LEN = 270). BlueZ `SOCK_SEQPACKET` **ne strip pas** ces headers — ils apparaissent dans les données retournées par `recv()`.

**Diagnostic :**
```python
# Premiers octets reçus : 00 01 7b 0a 20 20 22 73
#                          ^^^^^  = SDU_LEN (256 en LE16)
#                                ^^^^^^^^^^^^^^^^ = début du JSON "{..."
# Offset 258 : 00 01 65 79 66 69 6c 65
#              ^^^^^  = SDU_LEN du 2ème SDU
```

**Correction :** Parser les SDU_LEN headers dans le flux brut reçu :
```python
# Assembler tous les recv() bruts
all_raw = b"".join(raw_chunks)

# Parser les SDUs : [SDU_LEN LE16][data]
received = b""
pos = 0
while pos + 2 <= len(all_raw) and len(received) < obj_size_cur:
    sdu_len = struct.unpack("<H", all_raw[pos:pos + 2])[0]
    pos += 2
    received += all_raw[pos:pos + sdu_len]
    pos += sdu_len

# Tronquer à la taille annoncée
received = received[:obj_size_cur]
```

**Fichier :** `tools/ots_transfer.py`

---

### 19. OLCP GoTo First sélectionne le Directory Listing Object

**Symptôme :** `receive_file()` lit toujours "Directory" (29 bytes) au lieu de l'objet envoyé par `sotp-send`.

**Cause :** Le premier objet dans la liste OTS (ID 0x000000000000) est le **Directory Listing Object** (spec OTS §4.5.1), créé automatiquement par Zephyr quand `CONFIG_BT_OTS_DIR_LIST_OBJ=y`. OLCP GoTo First le sélectionne systématiquement.

**Correction :** Après OLCP GoTo First, envoyer un OLCP GoTo Next pour sauter le Directory :
```python
# GoTo First
await client.write_gatt_char(UUID_OLCP, struct.pack("<B", 0x01), response=True)
await olcp_event.wait()

# Skip Directory → GoTo Next
await client.write_gatt_char(UUID_OLCP, struct.pack("<B", 0x02), response=True)
await olcp_event.wait()
```

**Fichier :** `tools/ots_transfer.py`

---

### 20. OLCP write sans subscription aux indications → CCCD Improperly Configured

**Symptôme :** `BleakGATTError: GATT CCCD Improperly Configured` lors de l'écriture OLCP GoTo First.

**Cause :** La spec BLE OTS exige que le client souscrive aux indications OLCP (CCCD = 0x0002) avant d'écrire sur le point de contrôle. Sans souscription, le serveur rejette l'écriture.

**Correction :** Souscrire aux indications OLCP avant le premier write :
```python
await client.start_notify(UUID_OLCP, olcp_indication_handler)
# Puis seulement :
await client.write_gatt_char(UUID_OLCP, olcp_first, response=True)
```

**Fichier :** `tools/ots_transfer.py`

---

### 21. sotp-bridge à 1 Mbaud vs NINA à 115200

**Symptôme :** `sotp-bridge` ne reçoit pas les trames SOTP OBJ_FROM_BLE que la NINA envoie. Les fichiers envoyés depuis Ubuntu n'arrivent pas sur le filesystem iMX6. Le `journalctl` affiche `opened /dev/ttymxc7 at 1000000 baud (CRTSCTS)`.

**Cause :** Le `sotp-bridge.c` avait été compilé avec `B1000000` + `CRTSCTS` lors d'une tentative de passage à 1 Mbaud. La NINA avait été revert à 115200 sans flow control, mais pas le sotp-bridge.

**Correction :** Remettre le sotp-bridge à 115200 sans flow control :
```c
cfsetospeed(&tty, B115200);
cfsetispeed(&tty, B115200);
cfmakeraw(&tty);
tty.c_cflag &= ~CRTSCTS;  /* pas de flow control */
```

**Fichier :** `layers/STIMIODEV/SIOT10069-stimio-meta-app/recipes-connectivity/sotp-bridge/files/sotp-bridge.c`

---

### 22. Modem Quectel USB cause des CRC mismatch UART

**Symptôme :** CRC mismatch intermittents sur les trames SOTP de 30 KB entre NINA et iMX6. Les chunks 0 sont perdus, les suivants arrivent avec "no active transfer". Le modem USB Quectel EG91 se reconnecte (`usb 1-1: new high-speed USB device`) pendant les transferts.

**Cause :** La reconnexion du modem USB génère des interruptions kernel intenses qui perturbent le driver UART iMX6 (`ttymxc7`). À 115200 baud, une trame de 30 KB prend ~2.7s → large fenêtre d'interférence.

**Contournement :**
```bash
# Arrêter le service PPP avant les transferts BLE
systemctl stop ppp
```

**Amélioration future :** Passer à 1 Mbaud + hw-flow-control (nécessite investigation DTS iMX6 pour router RTS/CTS vers la NINA), ce qui réduirait le temps de transfert UART à ~0.3s par chunk.

---

### 23. ACK confusion dans multi-chunk FILE_REQUEST

**Symptôme** : le sotp-bridge envoie le chunk 0 puis timeout en attente de l'ACK. Le CRC mismatch `recv=0xaaf9 calc=0xc2f9` est reproductible à chaque fois.

**Cause** : Quand le sotp-bridge envoie un OBJ_TO_BLE, la NINA l'ajoute au pool OTS et envoie immédiatement un ACK (type 0x03). Le `wait_for_ack` du sotp-bridge devait attendre le DELETE_ACK (type 0x07) mais recevait le ACK normal (0x03) qui arrivait pendant le flush, et les octets résiduels dans le buffer UART corrompaient le CRC.

**Correction** :
1. Introduire `SOTP_TYPE_DELETE_ACK` (0x07) distinct de `SOTP_TYPE_ACK` (0x03)
2. Vider le buffer UART (flush 50ms) avant d'attendre le DELETE_ACK
3. Réinitialiser la state machine SOTP RX dans `wait_for_ack`

**Fichiers** : `uart_relay.h`, `uart_relay.c`, `sotp-bridge.c`

---

### 24. bt_ots_obj_delete passe toujours conn=NULL au callback obj_deleted

**Symptôme** : le DELETE_ACK n'est jamais envoyé car le test `if (conn)` dans `obj_deleted` est toujours faux.

**Cause** : Zephyr `bt_ots_obj_delete()` (ots.c:420) appelle `obj_deleted(ots, NULL, obj->id)` avec conn=NULL **toujours**, que la suppression vienne d'un OACP Delete client ou d'un appel interne.

**Correction** : utiliser un flag `cleaning_up` au lieu de tester `conn`. Le flag est mis à `true` pendant `ots_handler_cleanup_on_disconnect()` et `false` sinon. `obj_deleted` envoie le DELETE_ACK quand `!cleaning_up`.

**Fichier** : `ots_handler.c`

---

### 25. BlueZ SOCK_SEQPACKET limite les crédits L2CAP à ~16

**Symptôme** : un objet OTS de 30 KB n'est reçu qu'à hauteur de ~6.7 KB (26 SDUs × 258 bytes). Le `recv()` retourne EOF après 6708 bytes.

**Cause** : BlueZ `SOCK_SEQPACKET` pour L2CAP CoC n'accorde que ~16 crédits initiaux au récepteur. Chaque crédit permet un SDU de ~258 bytes (MPS 245 + SDU_LEN 2 + overhead). 16 × 258 ≈ 4128 bytes maximum recevable. Les crédits supplémentaires ne sont pas accordés tant que le SDU complet n'est pas consommé par `recv()`, mais le SDU fait 30 KB et ne peut jamais être "complet" avec seulement 16 crédits.

**Contournement** : réduire les chunks OTS à 3800 bytes (15 SDUs, dans les 16 crédits).

**Amélioration future** : utiliser `SOCK_STREAM`, l'API D-Bus BlueZ, ou patcher le kernel.

**Fichier** : `sotp-bridge.c` (OTS_OBJ_MAX_SIZE)

---

## Problème résolu : L2CAP CoC impossible depuis socket raw Python

### Description

Ce problème a été le **bloquant principal** avant la solution `libc.connect()` avec `sockaddr_l2`. Le canal L2CAP CoC PSM 0x0025 ne s'ouvre jamais côté NINA, malgré :
- NINA confirmée fonctionnelle (PSM 0x0025 enregistré, `l2cap_accept` visible en logs DBG)
- Connexion GATT Bleak opérationnelle (OACP Create, Name Write OK)
- Handle HCI connexion existante : `2048` (confirmé via `hcitool con`)

### Cause identifiée

**BlueZ 5.72 refuse de multiplexer un canal L2CAP CoC raw sur une connexion BLE déjà gérée par `bluetoothd`.**

- Les sockets `AF_BLUETOOTH / SOCK_SEQPACKET / BTPROTO_L2CAP` sont interceptés par `bluetoothd`
- BlueZ ne transmet **jamais** le `LE Credit Based Connection Request` au niveau HCI
- Confirmé par `btmon` : aucun paquet L2CAP CoC dans les traces HCI
- `l2test -V le_public` donne `Connection refused (111)` → BlueZ tente mais la NINA reçoit la demande trop tard (ou connexion BLE déjà terminée)
- `l2test -V le_random` donne `EINPROGRESS` mais aucune connexion côté NINA

### Tentatives échouées

| Tentative | Résultat |
|-----------|----------|
| `sock.connect((addr, PSM))` Python tuple | `Host is down (112)` |
| `libc.connect()` avec `BDADDR_LE_RANDOM` | Timeout — BlueZ ignore |
| `libc.connect()` avec `BDADDR_LE_PUBLIC` | `Host is down (112)` |
| `sock.bind((local_addr, 0))` | `Invalid argument (22)` |
| `setsockopt BT_MODE LE_FLOWCTL` | `Invalid argument (22)` sur tous les formats |
| `setsockopt BT_RCVMTU` | `Invalid argument (22)` |
| `l2test -V le_public` | `Connection refused (111)` |
| `l2test -V le_random` | `EINPROGRESS` — NINA ne reçoit rien |

### Solution en cours : API BlueZ D-Bus

BlueZ expose un mécanisme `org.bluez.Profile1` + `ProfileManager1.RegisterProfile()` qui permet à une application de recevoir un **file descriptor** de socket L2CAP CoC via le callback `NewConnection()`.

Implémentation dans `ots_send.py` :
1. Créer un objet D-Bus `OTSProfile` qui implémente `org.bluez.Profile1`
2. `RegisterProfile()` avec `PSM=0x0025` et `UUID=0x1825`
3. Appeler `Device1.ConnectProfile(UUID)` sur le device déjà connecté
4. BlueZ ouvre le canal L2CAP CoC et appelle `NewConnection(fd)`
5. Utiliser ce `fd` comme socket pour envoyer les données

**État :** Implémenté dans `ots_send.py` (remplacement complet de la section socket raw). **Non encore testé** — à valider au prochain test.

---

## Logs de référence

### Log RTT NINA — état normal au démarrage
```
[DBG] bt_l2cap: bt_l2cap_server_register: PSM 0x0025
[DBG] bt_ots: bt_gatt_ots_l2cap_init: Initialized OTS L2CAP
[INF] main: Bluetooth initialized
[INF] ots_handler: OTS server ready (pool: 3 objects × 32 KB each)
```

### Log RTT NINA — séquence GATT OK (sans L2CAP CoC)
```
[INF] main: Connected: A0:B3:39:B4:00:95 (public)
[INF] ots_handler: obj_created: id=0x000000000100 idx=0 size_alloc=17 name='' (pool: 1/3)
[INF] ots_handler: obj_name_written: id=0x000000000100 '' → 'test.bin'
[INF] main: Disconnected: A0:B3:39:B4:00:95 (public), reason=19 ()
[ERR] main: bt_le_adv_start after disconnect failed: -12
```

### Log `hcitool con` pendant connexion Bleak
```
Connections:
    < LE DE:CA:F1:5B:AD:11 handle 2048 state 1 lm CENTRAL
```

---

## Commandes utiles

### Rebuild + flash firmware NINA
```bash
cd /home/quentin/TRAVAIL/git_soft_dev/ZephyrHCI_Railnet200/zephyrproject
west build -d build_ots --pristine=always -b railnet200_nina_b301/nrf52840 \
    zephyr/samples/bluetooth/peripheral_ots_railnet
west flash -d build_ots
```

### Lancer le script de transfert
```bash
# Doit être lancé avec le Python SYSTÈME (pas le venv) car dbus n'est pas dans le venv
sudo python3 tools/ots_send.py DE:CA:F1:5B:AD:11 /tmp/test.bin
```

### Capturer les traces HCI
```bash
sudo btmon 2>&1 | tee /tmp/btmon.log
```

### Vérifier les connexions BLE actives
```bash
hcitool con
```

### Vérifier le status BlueZ
```bash
systemctl status bluetooth
journalctl -u bluetooth -n 20
```

### Vérifier le mode expérimental BlueZ
```bash
cat /etc/systemd/system/bluetooth.service.d/override.conf
# Doit contenir : ExecStart=/usr/sbin/bluetoothd --experimental
```

---

## Architecture du code

```
ots_send.py
├── Bleak (GATT)
│   ├── Scan + connexion BLE
│   ├── Subscribe OACP indications (0x2AC5)
│   ├── OACP Create (opcode 0x01)
│   ├── Write Object Name (0x2ABE)
│   └── OACP Write (opcode 0x06)  ← nécessite canal L2CAP ouvert
└── D-Bus BlueZ (L2CAP CoC)
    ├── RegisterProfile(PSM=0x0025, UUID=0x1825)
    ├── ConnectProfile() → bluetoothd initie le canal
    ├── NewConnection(fd) ← bluetoothd appelle notre callback
    └── sock.send(data) sur le fd reçu

peripheral_ots_railnet/ (Zephyr NINA-B301)
├── main.c          — advertising, connexion BLE, thread stats
├── ots_handler.c   — pool d'objets, callbacks OTS, relay UART
└── uart_relay.c    — protocole SOTP vers iMX6 /dev/ttymxc7
```
