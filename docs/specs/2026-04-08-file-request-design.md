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

## Gestion d'erreurs

| Cas | Côté sotp-bridge | Côté Python |
|-----|-------------------|-------------|
| Fichier introuvable | NACK 0x10 | Timeout 15s → message d'erreur |
| Fichier > 32 KB | NACK 0x11 | Timeout 15s → message d'erreur |
| Erreur I/O | NACK 0x12 | Timeout 15s → message d'erreur |
| Pool OTS plein (NINA) | NINA NACK au sotp-bridge | Timeout 15s |
| Déconnexion BLE | N/A | Timeout naturel |

Python détecte tous les échecs par timeout car il n'y a pas de canal de retour SOTP→BLE pour les NACK.

## Ce qui ne change pas

- Le send (Ubuntu → iMX6) reste identique
- Le receive existant (`receive` sous-commande) reste disponible pour les objets envoyés manuellement via `sotp-send`
- Le format SOTP OBJ_TO_BLE `[NAME_LEN][NAME][DATA]` est réutilisé tel quel
- Le sotp-bridge continue de recevoir les OBJ_FROM_BLE et de les écrire en fichiers

## Évolution future

- Support fichiers > 32 KB : le sotp-bridge enverrait plusieurs OBJ_TO_BLE avec un mécanisme de chunking (header chunk_idx/chunk_total, comme le send)
- Canal de retour pour les erreurs : caractéristique GATT custom ou réponse dans le pool OTS
