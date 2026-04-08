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

**Solution à implémenter :** Ajouter dans `ots_handler.c` une fonction `ots_handler_cleanup_on_disconnect()` appelée depuis `main.c::disconnected()` qui remet tous les slots `in_use=false` et `obj_count=0`.

---

## Problème en cours : L2CAP CoC impossible depuis socket raw Python

### Description

C'est le problème **bloquant actuel**. Le canal L2CAP CoC PSM 0x0025 ne s'ouvre jamais côté NINA, malgré :
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
