# SIOT10XXX-ZephyrBLE — Firmware BLE NINA-B301 pour RailNet200

Firmware Zephyr pour le module u-blox NINA-B301 (nRF52840) embarqué dans le gateway RailNet200 (Naonext/STIMIO).

## Architecture matérielle

```
┌─────────────────────────────────────────────────────────┐
│                   RailNet200 Gateway                     │
│                                                          │
│  ┌──────────────┐    UART (ttymxc7)    ┌──────────────┐ │
│  │   iMX6 SoC   │◄───────────────────►│  NINA-B301   │ │
│  │  Linux/Yocto  │   1 Mbaud, 8N1     │  nRF52840    │ │
│  │              │                      │  Zephyr RTOS │ │
│  └──────────────┘                      └──────┬───────┘ │
│                                               │ BLE     │
└───────────────────────────────────────────────┼─────────┘
                                                │
                                    ┌───────────▼──────────┐
                                    │  PC / Smartphone     │
                                    │  (BLE OTS / NUS)     │
                                    └──────────────────────┘
```

## Prérequis

### Outils

- **Zephyr SDK** : v0.17.0+ ([installation](https://docs.zephyrproject.org/latest/develop/getting_started/index.html))
- **West** : `pip install west`
- **J-Link** : pour flasher la NINA via SWD (Segger J-Link ou compatible)
- **Python 3.10+** : pour les scripts de transfert BLE côté PC

### Cloner le projet

```bash
git clone --recurse-submodules git@github.com:QBossardStimio/SIOT10XXX-ZephyrBLE.git
cd SIOT10XXX-ZephyrBLE
```

### Initialiser l'environnement West

```bash
cd zephyrproject
west init -l zephyr    # utilise le manifest west.yml du submodule
west update            # clone bootloader, modules, tools (~6 GB)
west zephyr-export
pip install -r zephyr/scripts/requirements.txt
```

## Samples disponibles

### 1. `peripheral_ots_railnet` — Transfert de fichiers via BLE OTS

**Le sample principal.** Reçoit des fichiers via BLE OTS (L2CAP CoC) et les retransmet à l'iMX6 via UART en protocole SOTP.

```bash
cd zephyrproject
west build -d build_ots -b railnet200_nina_b301/nrf52840 \
    zephyr/samples/bluetooth/peripheral_ots_railnet -- \
    -DCONFIG_BT_L2CAP_SEG_RECV=y
west flash -d build_ots
```

**Fonctionnalités :**
- Pool de 4 objets BLE OTS (configurable)
- Réception par segments L2CAP (`CONFIG_BT_L2CAP_SEG_RECV`) — pas besoin de buffer SDU complet en RAM
- Relay UART vers iMX6 en protocole SOTP (AA 55 TYPE LEN PAYLOAD CRC16)
- Suppression d'objet différée via `k_work` (contourne le -EBUSY de la state machine OTS)
- Re-advertising automatique après déconnexion

**Test depuis Ubuntu :**

```bash
# Depuis le dossier racine du projet SIOT10066-stimio-linux-menus
cd tools
python3 ots_transfer.py
```

### 2. `peripheral_nus` — Bridge UART bidirectionnel via BLE NUS

Bridge bidirectionnel entre BLE NUS et UART iMX6. Permet d'envoyer/recevoir des données série via une app smartphone (nRF Connect, Serial Bluetooth Terminal...).

```bash
cd zephyrproject
west build -d build_nus -b railnet200_nina_b301/nrf52840 \
    zephyr/samples/bluetooth/peripheral_nus
west flash -d build_nus
```

**Fonctionnalités :**
- NUS RX (smartphone → NINA) : relayé vers UART iMX6
- UART RX (iMX6 → NINA) : accumulé avec fenêtre de 50ms, envoyé via NUS TX
- Chunks NUS de 20 bytes (MTU BLE minimal garanti)
- BD Address statique : `DE:CA:F1:5B:AD:00`

### 3. `hci_uart` — Contrôleur BLE HCI sur UART

Transforme la NINA en contrôleur BLE HCI piloté par l'iMX6 via UART. L'iMX6 exécute BlueZ et utilise la NINA comme radio BLE.

```bash
cd zephyrproject
west build -d build_hci -b railnet200_nina_b301/nrf52840 \
    zephyr/samples/bluetooth/hci_uart
west flash -d build_hci
```

**Fonctionnalités :**
- BD Address statique : `DE:CA:F1:5B:AD:00`
- Thread de statistiques HCI (RX/TX packets, RX bytes) toutes les 10s
- Console Segger RTT (ne conflit pas avec UART HCI)

**Configuration côté iMX6 :**
```bash
# Attacher le HCI UART
sudo btattach -B /dev/ttymxc7 -S 1000000 -P h4 &
# Vérifier
hciconfig hci1 up
hcitool -i hci1 cmd 0x03 0x0009  # Read BD_ADDR
```

### 4. `echo_bot` — Test UART bas niveau

Envoie un pattern `0xDECAFBAD` en continu sur l'UART et affiche les octets reçus en hex. Utile pour valider la liaison UART entre NINA et iMX6 sans la couche BLE.

```bash
cd zephyrproject
west build -d build_echo -b railnet200_nina_b301/nrf52840 \
    zephyr/samples/drivers/uart/echo_bot
west flash -d build_echo
```

## Board : `railnet200_nina_b301`

Définition custom pour le module NINA-B301 tel que câblé dans le RailNet200 :

| Signal | Fonction | Pin nRF52840 |
|--------|----------|-------------|
| UART TX | → iMX6 RX (ttymxc7) | Voir DTS pinctrl |
| UART RX | ← iMX6 TX (ttymxc7) | Voir DTS pinctrl |
| RESET | Contrôlé par iMX6 GPIO503 | N/A (hardware) |
| SWD | J-Link debug | SWDIO/SWCLK |

**Console :** Segger RTT (pas UART — l'UART est utilisé pour les données).

Fichiers : `zephyr/boards/u-blox/railnet200_nina_b301/`

## Modifications du stack Zephyr

Ce fork contient des modifications dans le stack BLE Zephyr (`subsys/bluetooth/`) :

### OTS L2CAP (`subsys/bluetooth/services/ots/`)
- Support `CONFIG_BT_L2CAP_SEG_RECV` : réception par segments sans buffer SDU complet
- Callback `seg_recv` avec gestion des crédits L2CAP par PDU
- Calcul MPS optimal pour DLE (245 bytes)
- Extension Kconfig `RX_MTU` jusqu'à 65533 avec SEG_RECV

### OACP Write (`subsys/bluetooth/services/ots/ots_oacp.c`)
- Refactoring : `oacp_write_common()` partagé entre mode classique et SEG_RECV
- Nouveau callback `oacp_write_seg_cb` pour écriture par segments

### Host diagnostics (`subsys/bluetooth/host/`)
- `l2cap.c` : LOG_DBG dans `seg_recv` (sdu_len, offset, seg_len)
- `conn.c` : détails dans l'erreur "Not enough buffer space" (tailroom, rx_len, rx_size)

## Branches

```
master ← base
  └── feature.railnet200-ots-file-transfer  ← transfert fichier BLE OTS fonctionnel
        └── feature.railnet200-ots-directory     ← (futur) envoi de dossiers
              └── feature.railnet200-ots-large-files  ← (futur) gros fichiers
```

Les branches sont chaînées : chaque feature se rebase sur la précédente.

## Monitoring RTT (debug firmware)

```bash
# Terminal 1 : JLinkRTTLogger ou Segger RTT Viewer
JLinkRTTLogger -Device NRF52840_XXAA -if SWD -Speed 4000 -RTTChannel 0 /dev/stdout
```

## Liens

- **Repo principal RailNet (Yocto)** : `SIOT10066-stimio-linux-menus`
- **Meta-app (sotp-bridge)** : `SIOT10069-stimio-meta-app` — côté iMX6, reçoit les fichiers SOTP
- **Fork Zephyr** : [QBossardStimio/zephyr_4_3_0](https://github.com/QBossardStimio/zephyr_4_3_0)
