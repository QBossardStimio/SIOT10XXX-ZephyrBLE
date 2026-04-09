# FILE_REQUEST — Récupérer un fichier iMX6 via BLE OTS

## Objectif

Permettre au script Python (`ots_transfer.py`) de demander un fichier spécifique depuis le filesystem iMX6 par son chemin. Le sotp-bridge gère la requête automatiquement — plus besoin d'arrêter le service et de lancer `sotp-send` manuellement.

## Contraintes

- Fichier réponse max 32 KB (un seul objet OTS dans le pool NINA)
- UART 115200, pas de flow control
- Le sotp-bridge reste l'unique processus sur l'UART

## Flux de données

```
Python (Ubuntu)                    NINA-B301 (Zephyr)              iMX6 (sotp-bridge)
     |                                  |                               |
     |-- OACP Create (size=pathlen) --> |                               |
     |-- Write Name "FILE_REQ" -------> |                               |
     |-- OACP Write (path bytes) -----> |                               |
     |       [L2CAP CoC send]          |                               |
     |                                  |-- SOTP FILE_REQUEST --------> |
     |                                  |   (type=0x06, payload=path)   |
     |                                  |                               |-- fopen(path)
     |                                  |                               |-- fread()
     |                                  | <-- SOTP OBJ_TO_BLE ---------|
     |                                  |   (type=0x02, [name][data])   |
     |                                  |                               |
     |                                  |-- ots_handler_add_object() -->|
     |                                  |   (objet dans le pool OTS)    |
     |                                  |                               |
     |-- OLCP poll (500ms) -----------> |   (cherche objet != Directory)|
     |-- Read Object Name / Size -----> |                               |
     |-- OACP Read -------------------> |                               |
     |       [L2CAP CoC recv]          |                               |
     |                                  |                               |
     DONE
```

## Nouveau type SOTP

```c
#define SOTP_TYPE_FILE_REQUEST  0x06U
```

Payload : le chemin du fichier en UTF-8 brut (pas de null-terminator, longueur = LEN du header SOTP).

## Modifications par composant

### 1. NINA — `uart_relay.h`

Ajouter la constante :
```c
#define SOTP_TYPE_FILE_REQUEST  0x06U
```

### 2. NINA — `uart_relay.c`

Nouvelle fonction publique :
```c
int uart_relay_send_file_request(const uint8_t *path, size_t len);
```

Envoie une trame SOTP type=0x06 avec le path en payload. Pas de blocage (même pattern que `uart_relay_send_object`).

### 3. NINA — `ots_handler.c` (obj_write callback)

Quand `remaining == 0` (objet complet) :
- Si le nom de l'objet est `"FILE_REQ"` → appeler `uart_relay_send_file_request(data, written_len)` au lieu de `uart_relay_send_object()`
- Sinon → comportement actuel (`uart_relay_send_object` type 0x01)

Dans les deux cas, supprimer l'objet via `k_work_submit(&delete_work)`.

### 4. sotp-bridge — constantes

Ajouter :
```c
#define SOTP_TYPE_FILE_REQUEST  0x06U
#define SOTP_ERR_NOT_FOUND      0x10U
#define SOTP_ERR_TOO_LARGE      0x11U
#define SOTP_ERR_IO             0x12U
#define FILE_REQUEST_MAX_SIZE   (32 * 1024)
```

### 5. sotp-bridge — `sotp_dispatch()`

Nouveau case `SOTP_TYPE_FILE_REQUEST` :
1. Extraire le path depuis le payload (longueur = `len`, pas de null-terminator → copier et ajouter `\0`)
2. `fopen(path, "rb")` → si erreur : NACK `SOTP_ERR_NOT_FOUND`
3. `fstat()` → si taille > 32 KB : NACK `SOTP_ERR_TOO_LARGE`
4. `fread()` → si erreur : NACK `SOTP_ERR_IO`
5. Construire le payload OBJ_TO_BLE : `[NAME_LEN uint8][basename][DATA]`
6. Envoyer via `sotp_send_frame(SOTP_TYPE_OBJ_TO_BLE, payload, len)`
7. Envoyer ACK (ou pas — le OBJ_TO_BLE sert de réponse implicite)

### 6. Python — `ots_transfer.py`

Nouvelle sous-commande `request` :
```bash
python3 ots_transfer.py DE:CA:F1:5B:AD:11 request --path /etc/stimio/config.json -o /tmp/config.json
```

Séquence :
1. Scan BLE + connexion
2. Subscribe OACP + OLCP indications
3. OACP Create (size = len(path_bytes))
4. Write Object Name = `"FILE_REQ"`
5. OACP Write (offset=0, len=len(path_bytes))
6. Envoyer le path via L2CAP CoC
7. Attendre l'indication OACP Write Success
8. Fermer le canal L2CAP
9. **Polling OLCP** : boucle toutes les 500ms (timeout 15s)
   - OLCP GoTo First → skip Directory → OLCP GoTo Next
   - Si un objet non-Directory existe → lire son nom et sa taille
   - Si pas d'objet → sleep 500ms et retry
10. Ouvrir L2CAP CoC + OACP Read
11. Recevoir les données (strip SDU_LEN headers)
12. Tronquer à `obj_size_cur` et sauvegarder

## Multi-chunk (implémenté)

Les fichiers > 3.8 KB sont découpés en chunks de 3800 bytes max. Chaque chunk est envoyé comme un objet OTS séparé avec le header `[chunk_idx 2B][chunk_total 2B][path_len 2B][data]` (même format que le send).

### Synchronisation multi-chunk

```
sotp-bridge                    NINA                         Python
    |-- OBJ_TO_BLE chunk 0 -->|                              |
    |                          |-- add to OTS pool           |
    |<-- ACK (type 0x03) -----|                              |
    |   (flushed by wait_for_ack)                            |
    |                          |                  <-- OLCP poll
    |                          |                  <-- OACP Read
    |                          |-- L2CAP data -->|
    |                          |                  <-- OACP Delete
    |                          |-- obj_deleted               |
    |<-- DELETE_ACK (0x07) ----|                              |
    |-- OBJ_TO_BLE chunk 1 -->|                              |
    ...
```

### Nouveau type SOTP

```c
#define SOTP_TYPE_DELETE_ACK  0x07U
```

La NINA envoie un DELETE_ACK (type 0x07) quand un objet est supprimé par un client BLE (OACP Delete). Le sotp-bridge attend ce DELETE_ACK avant d'envoyer le chunk suivant. Le ACK normal (type 0x03) envoyé par `ots_handler_add_object_from_uart` est flushed par `wait_for_ack` pour éviter la confusion.

### Limite de taille chunk : 3800 bytes

BlueZ `SOCK_SEQPACKET` n'accorde que ~16 crédits L2CAP initiaux au récepteur. Chaque crédit permet un SDU de ~258 bytes. Total recevable = 16 × 256 = ~4096 bytes. Avec marge de sécurité → 3800 bytes par objet OTS.

## Gestion d'erreurs

| Cas | Côté sotp-bridge | Côté Python |
|-----|-------------------|-------------|
| Fichier introuvable | NACK 0x10 | Timeout 30s → message d'erreur |
| Fichier > 2 MB | NACK 0x11 | Timeout 30s → message d'erreur |
| Erreur I/O | NACK 0x12 | Timeout 30s → message d'erreur |
| Pool OTS plein (NINA) | NINA NACK au sotp-bridge | Timeout 30s |
| Déconnexion BLE | N/A | Timeout naturel |

Python détecte tous les échecs par timeout car il n'y a pas de canal de retour SOTP→BLE pour les NACK.

## Ce qui ne change pas

- Le send (Ubuntu → iMX6) reste identique
- Le receive existant (`receive` sous-commande) reste disponible pour les objets envoyés manuellement via `sotp-send`
- Le format SOTP OBJ_TO_BLE `[NAME_LEN][NAME][DATA]` est réutilisé tel quel
- Le sotp-bridge continue de recevoir les OBJ_FROM_BLE et de les écrire en fichiers

## Résultats de test

| Fichier | Taille | Chunks | Durée | MD5 |
|---------|--------|--------|-------|-----|
| config.json | 522 B | 1 | ~35s | PASS |
| test_100k.bin | 100 KB | 27 | ~15 min | PASS |

## Optimisations futures (plan)

### 1. Augmenter les crédits L2CAP BlueZ (priorité haute)

**Problème** : BlueZ SOCK_SEQPACKET n'accorde que ~16 crédits L2CAP initiaux, limitant les chunks à ~3.8 KB. Avec 30 KB par chunk, le transfert 100 KB prendrait 4 chunks au lieu de 27.

**Pistes** :
- Configurer `BT_RCVMTU` avant `connect()` (testé, pas d'effet sur SEQPACKET)
- Utiliser `SOCK_STREAM` au lieu de `SOCK_SEQPACKET` (BlueZ gère les crédits en flux continu)
- Patcher le kernel BlueZ pour augmenter les crédits initiaux L2CAP CoC
- Utiliser l'API D-Bus BlueZ `AcquireNotify` / `AcquireWrite` au lieu de raw sockets

**Impact** : réduirait le nombre de chunks de ~27 à ~4 pour 100 KB, et de ~553 à ~35 pour 2 MB.

### 2. Réduire le polling OLCP (priorité moyenne)

**Problème** : chaque chunk nécessite un polling OLCP (GoTo First → GoTo Next → Read Name) avec 500ms entre les tentatives. Ça ajoute ~1-3s par chunk.

**Pistes** :
- Réduire l'intervalle de poll à 100ms
- Utiliser une caractéristique GATT custom pour notifier Python quand un objet est ajouté
- Pré-charger 2-3 chunks dans le pool OTS (pipeline)

### 3. Passer le baudrate UART à 1 Mbaud (priorité moyenne)

**Problème** : à 115200 baud, un chunk de 3.8 KB prend ~330ms sur l'UART. À 1 Mbaud, ce serait ~38ms.

**Prérequis** :
- Investiguer le DTS iMX6 pour router les pins RTS/CTS vers la NINA
- Activer `hw-flow-control` dans le DTS Zephyr NINA
- Réactiver les pins RTS/CTS dans le pinctrl
- Recompiler sotp-bridge avec `B1000000` + `CRTSCTS`

### 4. Canal de retour pour les erreurs (priorité basse)

**Problème** : Python ne peut pas savoir si le FILE_REQUEST a échoué (fichier introuvable, trop gros). Il doit attendre le timeout de 30s.

**Pistes** :
- Caractéristique GATT custom pour les erreurs SOTP
- Objet OTS "erreur" avec un nom spécial (ex: `ERR:NOT_FOUND`)
