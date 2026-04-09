# FILE_REQUEST — Plan d'optimisation du débit

## Contexte

Le FILE_REQUEST multi-chunk fonctionne mais est lent :
- 100 KB → 27 chunks × ~33s/chunk = **15 min**
- 2 MB → ~553 chunks → **~5h** (estimé)

Le bottleneck principal est la taille des chunks (3.8 KB) limitée par les crédits L2CAP BlueZ.

## Optimisations par priorité

### 1. Augmenter les crédits L2CAP BlueZ (priorité haute, gain ~8x)

**Problème** : BlueZ `SOCK_SEQPACKET` fixe `imtu=672` en dur. Cela donne ~16 crédits L2CAP initiaux, limitant la réception à ~4 KB par objet OTS. Avec 30 KB par chunk, on aurait 4 chunks au lieu de 27 pour 100 KB.

**Testé (2026-04-09, kernel 6.17)** :

| Tentative | Résultat |
|-----------|----------|
| `setsockopt(BT_RCVMTU, 65535)` | `EINVAL` — non supporté |
| `setsockopt(SO_RCVBUF, 1MB)` | Accepté mais aucun effet sur les crédits |
| `SOCK_STREAM` au lieu de `SOCK_SEQPACKET` | Se connecte mais `recv()` retourne 0 bytes |
| `setsockopt(BT_MODE, 3)` | `EINVAL` |

**Pistes non testées** :

1. **Patch kernel** `net/bluetooth/l2cap_sock.c` — modifier `l2cap_sock_init()` pour augmenter `chan->imtu` de 672 à 32768
2. **API D-Bus BlueZ** — utiliser `org.bluez.GattCharacteristic1.AcquireNotify` qui ouvre un fd L2CAP avec MTU négocié plus grand
3. **BlueZ `--experimental`** + `bt_shell` pour configurer les crédits L2CAP dynamiquement
4. **Upgrade kernel** — vérifier si les kernels > 6.17 permettent `setsockopt(BT_RCVMTU)` sur SEQPACKET

**Impact** :

| Fichier | Actuel (3.8 KB chunks) | Après (30 KB chunks) |
|---------|------------------------|----------------------|
| 100 KB | 15 min (27 chunks) | ~2 min (4 chunks) |
| 2 MB | ~5h (553 chunks) | ~40 min (70 chunks) |

### 2. UART 1 Mbaud + hw-flow-control (priorité moyenne, gain ~2x)

**Problème** : à 115200 baud, un chunk de 3.8 KB prend ~330ms sur l'UART. Le round-trip (envoi + ACK retour) prend ~700ms.

**Prérequis** :
- Investiguer le DTS iMX6 pour vérifier que les pins RTS/CTS sont routés vers la NINA (schémas RailNet200 v1.1/v1.3)
- Activer `hw-flow-control` dans le DTS Zephyr NINA (overlay)
- Réactiver `NRF_PSEL(UART_RTS/CTS)` dans le pinctrl
- Recompiler sotp-bridge avec `B1000000` + `CRTSCTS`
- Tester la stabilité (le 1 Mbaud sans flow control avait échoué)

**Impact** : temps UART divisé par ~8, mais le BLE reste le bottleneck principal. Gain global ~2x.

### 3. Pipeline 2-3 chunks d'avance (priorité basse, gain ~2-3x)

**Problème** : le sotp-bridge envoie un chunk puis attend le DELETE_ACK avant d'envoyer le suivant. Pendant ce temps, l'UART et le pool OTS NINA sont idle.

**Design** :
- Le sotp-bridge envoie 2-3 chunks d'avance (le pool NINA fait 4 slots)
- Python lit et supprime les objets au fur et à mesure
- Le sotp-bridge écoute les DELETE_ACK et maintient 2-3 chunks dans le pipeline

**Complexité** : gestion du backpressure, risque de pool saturation si Python est lent.

### 4. Canal de retour pour les erreurs (priorité basse)

**Problème** : quand un FILE_REQUEST échoue (fichier introuvable), Python doit attendre le timeout de 30s.

**Pistes** :
- Objet OTS "erreur" avec un nom spécial (ex: `ERR:NOT_FOUND`)
- Caractéristique GATT custom pour les notifications d'erreur SOTP

## Temps estimés après optimisations combinées

| Fichier | Actuel | #1 seul | #1 + #2 | #1 + #2 + #3 |
|---------|--------|---------|---------|--------------|
| 522 B | 35s | 35s | 35s | 35s |
| 100 KB | 15 min | ~2 min | ~1 min | ~30s |
| 2 MB | ~5h | ~40 min | ~20 min | ~8 min |
