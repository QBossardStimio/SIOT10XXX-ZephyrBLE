# UART RTS/CTS Hardware Flow Control — iMX6 ↔ NINA-B301 Debug Log

## Contexte

Activation du hardware flow control (RTS/CTS) sur la liaison UART entre l'iMX6UL (host Linux, `/dev/ttymxc7`) et le NINA-B301 (nRF52840, Zephyr UART0) de la carte RailNet200.

**Objectif final :** passer la liaison UART à 1 Mbaud avec RTS/CTS pour permettre le fonctionnement concurrent avec le service PPP (modem Quectel USB).

## Erreur de câblage critique — Inversion RTS/CTS dans les noms de nets

**Source :** schéma STO06_Carte_Railnet_C1.pdf, sheets 5 (Module CPU) et 19 (Bluetooth).

Le schéma nomme les signaux du point de vue **iMX6** (CTS = entrée iMX6, RTS = sortie iMX6), mais le câblage physique vers la NINA est **inversé par rapport à ce que ces noms suggèrent en terminologie UART standard**.

### Câblage physique réel (vérifié par RTT + devmem2)

```
iMX6 pad              ball   fonction iMX6       net PCB            NINA-B301 pin
─────────────────────  ────   ──────────────────  ────────────────   ────────────────────────
ENET2_TX_DATA1         53     UART8_DCE_TX (out)  BLE_UART_TX        pin 23 (P0.29) = NINA RXD
ENET2_TX_EN            54     UART8_DCE_RX (in)   BLE_UART_RX        pin 22 (P1.13) = NINA TXD
ENET2_TX_CLK           55     UART8_DCE_CTS (out) BLE_UART_CTS       pin 20 (P0.31) = NINA CTS
ENET2_RX_ER            58     UART8_DCE_RTS (in)  BLE_UART_RTS       pin 21 (P1.12) = NINA RTS
```

### Point clé : sémantique DCE_CTS et DCE_RTS dans le kernel iMX6

Les noms des macros kernel `DCE_CTS` et `DCE_RTS` sont **contre-intuitifs** :

| Macro kernel | input_reg | Direction physique | Signal UART réel | Rôle |
|---|---|---|---|---|
| `UART8_DCE_CTS` | 0x0000 (pas d'input routing) | **Sortie** | RTS output du contrôleur | iMX6 dit "envoie-moi des données" |
| `UART8_DCE_RTS` | 0x0658 (input routing) | **Entrée** | CTS input du contrôleur | Périphérique dit "tu peux m'envoyer" |

Vérification par registre `UCR2` (0x02024084) avec `crtscts` activé :
- UCR2 = 0x302F → bit 13 (CTSC) = 1, bit 14 (IRTS) = 0 → flow control HW actif
- Le signal RTS sort sur le pad `DCE_CTS` (ball 55) → vérifié via RTT : P0.31 = LOW

### Erreur initiale : mapping NINA inversé

Le pinctrl NINA initial assignait :
```
UART_RTS → P0.31   (pin 20)   ← ERREUR : P0.31 reçoit le signal de ball 55, c'est une ENTREE
UART_CTS → P1.12   (pin 21)   ← ERREUR : P1.12 doit envoyer vers ball 58, c'est une SORTIE
```

Le mapping correct (validé par RTT dual-pin) :
```
UART_CTS → P0.31   (pin 20)   ← Reçoit le RTS iMX6 (ball 55 DCE_CTS output)
UART_RTS → P1.12   (pin 21)   → Envoie vers le CTS iMX6 (ball 58 DCE_RTS input)
```

**Validation RTT avec les deux mappings :**
```
Mapping INCORRECT (RTS=P0.31, CTS=P1.12) :
  P0.31(ball55/CTS)=0  P1.12(ball58/RTS)=1   ← P1.12 HIGH = NINA ne drive pas son RTS

Mapping CORRECT   (CTS=P0.31, RTS=P1.12) :
  P0.31(ball55/CTS)=0  P1.12(ball58/RTS)=0   ← Les deux LOW = flow control bidirectionnel OK
```

---

## Problèmes rencontrés

### 1. Pins RTS/CTS dans le pinctrl NINA bloquent la réception UART

**Date :** 2026-04-09

**Symptôme :** La NINA ne reçoit aucun octet de l'iMX6 via UART, même sans `hw-flow-control` activé dans le DTS Zephyr.

**Cause :** Le driver nRF52840 UARTE programme les registres `PSEL.CTS` et `PSEL.RTS` dès que les pins sont déclarées dans le pinctrl, même si la propriété `hw-flow-control` est absente du DTS. Quand `PSEL.CTS` pointe vers un GPIO non piloté, le UARTE bloque la transmission en attendant que CTS soit asserté.

**Correction :** Retirer les pins RTS/CTS du pinctrl NINA quand on ne veut pas de flow control.

**Leçon :** Sur nRF52840, **ne jamais déclarer les pins CTS/RTS dans le pinctrl si le flow control n'est pas réellement utilisé**. Le UARTE les programme quoi qu'il arrive.

---

### 2. Confusion sur la direction des macros DCE_CTS / DCE_RTS

**Date :** 2026-04-09

**Symptôme :** Après activation de `crtscts` côté iMX6 avec `DCE_CTS` sur ball 55 et `DCE_RTS` sur ball 58, le RTT NINA montre CTS non asserté.

**Cause :** L'hypothèse initiale était que `DCE_CTS` = entrée CTS et `DCE_RTS` = sortie RTS. C'est **faux**. Les macros iMX6 nomment les signaux d'après la **pin du DTE**, pas le signal interne du contrôleur :
- `DCE_CTS` : le pad où arrive le signal CTS **du DTE** → mais pour le contrôleur DCE, c'est sa **sortie** RTS
- `DCE_RTS` : le pad où arrive le signal RTS **du DTE** → mais pour le contrôleur DCE, c'est son **entrée** CTS

**Diagnostic :** Analyse des macros dans `imx6ul-pinfunc.h` :
```c
/* input_reg = 0x0000 → pas d'input routing → SORTIE */
#define MX6UL_PAD_ENET2_TX_CLK__UART8_DCE_CTS  0x00fc 0x0388 0x0000 1 0

/* input_reg = 0x0658 → input routing configuré → ENTREE */
#define MX6UL_PAD_ENET2_RX_ER__UART8_DCE_RTS   0x0100 0x038c 0x0658 1 1
```

**Leçon :** Toujours vérifier `input_reg` dans les macros iMX6 pour déterminer la direction réelle. `input_reg = 0x0000` = sortie, `input_reg != 0` = entrée.

---

### 3. Mode DTE complet casse TX/RX — double inversion des signaux

**Date :** 2026-04-10

**Symptôme :** Après passage en mode DTE complet (4 pins DTE + `fsl,dte-mode`), la communication UART est totalement cassée dans les deux directions.

**Cause :** `fsl,dte-mode` active le bit `UFCR_DCEDTE` (bit 6) qui inverse les signaux internes. Les macros DTE inversent les directions au niveau pad. Les deux inversions se compensent au niveau contrôleur, mais au niveau **physique** :
- `DTE_RX` sur `ENET2_TX_DATA1` configure le pad en entrée, mais la trace va vers NINA RXD (aussi entrée) → deux entrées face à face
- `DTE_TX` sur `ENET2_TX_EN` configure le pad en sortie, mais la trace va vers NINA TXD (aussi sortie) → deux sorties face à face

**Diagnostic :** `devmem2` confirme UFCR.DCEDTE=1 (0x0B41), URXD vide, RTT NINA ne montre aucun "RX:" reçu.

**Leçon :** `fsl,dte-mode` n'est pas juste un flag cosmétique. Ne l'utiliser que si le PCB est **réellement câblé en DTE** (TX CPU → TX périphérique, sans croisement).

---

### 4. Configuration hybride DCE/DTE ne fonctionne pas

**Date :** 2026-04-10

**Symptôme :** Avec `DCE_TX`/`DCE_RX` pour les données et `DTE_RTS`/`DTE_CTS` pour le flow control (sans `fsl,dte-mode`), TX/RX fonctionne mais le flow control ne fonctionne pas.

**Cause :** Sans `fsl,dte-mode` (DCEDTE=0), le contrôleur UART est en mode DCE. Il envoie son signal RTS sur le pad `DCE_RTS` et lit CTS sur `DCE_CTS`. Les macros DTE pour RTS/CTS muxent les pads vers des signaux que le contrôleur DCE **n'utilise pas** → le RTS n'est jamais drivé sur les pads DTE.

**Diagnostic :** Même avec `UCR2.CTSC=1` et `UCR2.IRTS=0` (flow control activé), le RTT montre que P1.12 reste HIGH car le signal sort sur ball 58 (`DCE_RTS`, entrée) et non sur ball 55 (`DTE_RTS`).

**Leçon :** Les macros DCE et DTE pour RTS/CTS ne sont **pas interchangeables**. Le contrôleur n'utilise que les pads correspondant à son mode (DCE ou DTE).

---

### 5. Inversion RTS/CTS dans le pinctrl NINA — cause racine

**Date :** 2026-04-10

**Symptôme :** Avec la configuration full DCE (4 macros DCE, sans `fsl,dte-mode`), l'iMX6 drive bien ball 55 (DCE_CTS = sortie RTS) vers LOW. Mais le RTT lisait **P1.12** (qui est connecté à **ball 58**, pas ball 55). Le signal ne pouvait pas être vu car on mesurait le mauvais pin.

**Cause :** Le pinctrl NINA assignait `UART_RTS` à P0.31 et `UART_CTS` à P1.12. D'après le schéma (STO06 sheet 19) :
- P0.31 (pin 20) est connecté à ball 55 via net `BLE_UART_CTS` → P0.31 **reçoit** un signal → doit être `UART_CTS` (entrée)
- P1.12 (pin 21) est connecté à ball 58 via net `BLE_UART_RTS` → P1.12 **envoie** un signal → doit être `UART_RTS` (sortie)

**Diagnostic :** Echo_bot modifié pour lire les deux pins GPIO :
```
P0.31(ball55/CTS)=0  P1.12(ball58/RTS)=0  ← Après correction : les deux LOW = OK
```

**Correction :** Inverser les assignments dans le pinctrl NINA :
```dts
/* AVANT (INCORRECT) */
psels = <NRF_PSEL(UART_RTS, 0, 31)>,   /* P0.31 = sortie RTS ? NON, c'est une entrée ! */
        <NRF_PSEL(UART_CTS, 1, 12)>;    /* P1.12 = entrée CTS ? NON, c'est une sortie ! */

/* APRES (CORRECT) */
psels = <NRF_PSEL(UART_CTS, 0, 31)>,   /* P0.31 = entrée CTS ← ball 55 iMX6 DCE_CTS out */
        <NRF_PSEL(UART_RTS, 1, 12)>;    /* P1.12 = sortie RTS → ball 58 iMX6 DCE_RTS in  */
```

**Leçon :** Les noms de nets PCB (`BLE_UART_CTS`, `BLE_UART_RTS`) sont du point de vue iMX6, pas de la NINA. Toujours vérifier le schéma sheet par sheet et valider avec des mesures physiques.

---

### 6. `fsl,dte-mode` absent du device-tree malgré la présence dans le DTS source

**Date :** 2026-04-10

**Symptôme :** `ls /proc/device-tree/.../serial@2024000/fsl,dte-mode` retourne "No such file". Fausse alerte — le chemin du node était incorrect.

**Cause :** Le node UART8 est sous `/soc/bus@2000000/spba-bus@2000000/serial@2024000/`, pas directement sous `/soc/bus@2000000/serial@2024000/`.

**Diagnostic :** `cat /proc/device-tree/aliases/serial7` → `/soc/bus@2000000/spba-bus@2000000/serial@2024000`

**Leçon :** Toujours utiliser `/proc/device-tree/aliases/serialN` pour trouver le chemin exact.

---

### 7. Filesystem /tmp plein empêche toute création de fichier

**Date :** 2026-04-10

**Symptôme :** `echo "test" > /tmp/file` retourne "write error: No space left on device". Aucune commande de diagnostic ne peut écrire ses résultats.

**Cause :** Le filesystem NAND (UBIFS rootfs) était plein à 100%. Le shell ne pouvait plus créer de fichiers ni écrire de logs.

**Correction :** Reboot physique de la carte. Après reboot, `/tmp` (tmpfs 243 MB) est vide.

**Leçon :** Surveiller l'espace disque avant les sessions de debug longues. Les logs kernel du modem Quectel qui se déconnecte/reconnecte en boucle (~13s) peuvent remplir les logs.

---

## Configuration finale validée

### iMX6 DTS (v1.1 et v1.3)

```dts
&uart8 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_uart8>;
    fsl,uart-has-rtscts;
    /* All DCE mode: DCE_CTS is output (ball 55), DCE_RTS is input (ball 58) */
    status = "okay";
};

pinctrl_uart8: uart8grp {
    fsl,pins = <
        MX6UL_PAD_ENET2_TX_DATA1__UART8_DCE_TX   0x1b0b1  /* iMX6 TX → NINA RXD (pin23/P0.29) */
        MX6UL_PAD_ENET2_TX_EN__UART8_DCE_RX      0x1b0b1  /* iMX6 RX ← NINA TXD (pin22/P1.13) */
        MX6UL_PAD_ENET2_TX_CLK__UART8_DCE_CTS    0x1b0b1  /* ball55 out → NINA CTS (pin20/P0.31) */
        MX6UL_PAD_ENET2_RX_ER__UART8_DCE_RTS     0x1b0b1  /* ball58 in  ← NINA RTS (pin21/P1.12) */
    >;
};
```

### NINA-B301 pinctrl (Zephyr)

```dts
uart0_default: uart0_default {
    group1 {
        psels = <NRF_PSEL(UART_RX,  0, 29)>,   /* pin 23 ← iMX6 TX */
                <NRF_PSEL(UART_TX,  1, 13)>,    /* pin 22 → iMX6 RX */
                <NRF_PSEL(UART_CTS, 0, 31)>,    /* pin 20 ← ball 55 iMX6 DCE_CTS out */
                <NRF_PSEL(UART_RTS, 1, 12)>;    /* pin 21 → ball 58 iMX6 DCE_RTS in  */
    };
};
```

### sotp-bridge (iMX6 Linux)

```c
tty.c_cflag |= CRTSCTS;   /* hardware flow control activé */
```

---

## Outils de diagnostic utilisés

| Outil | Usage |
|---|---|
| `devmem2 <addr> w` | Lecture registres UART8 et IOMUX sur iMX6 |
| `stty -F /dev/ttymxc7 -a` | Vérification config port série iMX6 |
| `JLinkRTTClient` | Lecture logs Zephyr en temps réel (RTT via J-Link) |
| `echo_bot` Zephyr modifié | Lecture GPIO P0.31 + P1.12 via registres nRF52840 |
| `/proc/device-tree/aliases/serialN` | Trouver le chemin exact du node UART |
| Schéma STO06_Carte_Railnet_C1.pdf | Vérification câblage physique (sheets 5 et 19) |

### Registres iMX6 UART8 utiles

| Registre | Adresse | Bits clés |
|---|---|---|
| UCR1 | 0x02024080 | bit 0: UARTEN |
| UCR2 | 0x02024084 | bit 13: CTSC (1=hw flow ctrl), bit 14: IRTS (0=use RTS) |
| UFCR | 0x02024090 | bit 6: DCEDTE (0=DCE, 1=DTE) |
| USR2 | 0x02024098 | bit 0: RDR, bit 3: TXFE |
| IOMUX UART8_RTS select | 0x020E0658 | sélection source entrée CTS |

### Registres nRF52840 GPIO

| Registre | Adresse | Usage |
|---|---|---|
| GPIO P0 IN | 0x50000510 | Lecture pins P0.0–P0.31 (bit N = pin N) |
| GPIO P1 IN | 0x50000810 | Lecture pins P1.0–P1.15 (bit N = pin N) |

---

## Chronologie

| Date | Action | Résultat |
|---|---|---|
| 2026-04-09 | Test echo_bot avec RTS/CTS dans pinctrl NINA sans hw-flow-control | NINA bloquée — PSEL.CTS attend CTS |
| 2026-04-09 | Retrait RTS/CTS du pinctrl NINA | TX/RX OK sans flow control |
| 2026-04-09 | Ajout `crtscts` côté iMX6 + tentative DTE_RTS/DTE_CTS | Signal mesuré sur mauvais pin |
| 2026-04-09 | Passage full DTE (4 pins DTE + `fsl,dte-mode`) | TX/RX cassé — double inversion |
| 2026-04-10 | Fix : DCE(TX/RX) + DTE(RTS/CTS) sans `fsl,dte-mode` | Flow control ne fonctionne pas (mauvais pads) |
| 2026-04-10 | Fix : full DCE (4 macros DCE) sans `fsl,dte-mode` | TX/RX OK, flow control OK dans les registres |
| 2026-04-10 | Analyse schéma STO06 sheets 5+19 | Découverte inversion RTS/CTS côté NINA |
| 2026-04-10 | Inversion pinctrl NINA : CTS↔P0.31, RTS↔P1.12 | **Flow control bidirectionnel validé** |
| 2026-04-10 | Test RTT dual-pin : P0.31=0, P1.12=0 | Les deux signaux assertés ✅ |
| 2026-04-10 | Test RX iMX6 avec `crtscts` : 0xAA reçus | Communication NINA→iMX6 avec flow control ✅ |
| 2026-04-10 | Suite OTS 4/4 à 115200 + RTS/CTS + PPP actif | Tous PASS, aucun CRC mismatch ✅ |
| 2026-04-10 | Passage à 1 Mbaud + RTS/CTS | Suite OTS 4/4 PASS, Send 1MB : 42s (vs 119s) ✅ |
