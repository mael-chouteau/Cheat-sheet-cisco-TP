# Guide de Sauvegarde et Restauration Cisco (Switch, Routeur, ASA)

Ce document est destiné aux techniciens pour la **remise en route d'équipements Cisco** dont la mémoire flash/disque interne est vide ou corrompue. Il détaille l'**export des fichiers** depuis un appareil source fonctionnel et l'**import/restauration** sur un équipement vierge ou bloqué en mode ROMMON via TFTP.

---

## 1. Prérequis Matériels et Logiciels

* **PC Technique :** Avec port Ethernet et port Série (ou adaptateur USB-Série).
* **Câblage :**
  * Câble Console (Bleu/RJ45-DB9).
  * Câble Ethernet RJ45 droit.

* **Logiciels :**
  * **Putty** (Connexion Série : 9600 bauds, 8 data bits, 1 stop bit, No parity).
  * **TFTPD64** : [Télécharger la version portable](https://github.com/PJO2/tftpd64/releases/download/v4.74/tftpd64_portable_v4.74.zip).



### Configuration Réseau du PC

L'interface Ethernet du PC doit être configurée en IP fixe pour agir comme serveur :

* **IP Address :** `10.30.1.100`
* **Subnet Mask :** `255.255.255.0`

### Configuration TFTPD64

1. Lancer `tftpd64.exe` (en administrateur si nécessaire).
2. Dans **Server interfaces**, sélectionner l'IP `10.30.1.100`.
3. Vérifier dans **Settings** que l'option "Bind TFTP to this address" correspond bien à votre carte réseau.
4. **Répertoire des fichiers :** Les fichiers à transférer doivent être dans le répertoire de travail de TFTPD64 (ou dans le répertoire TFTP configuré). Vérifier le chemin affiché dans la fenêtre. Éviter les espaces dans les noms de fichiers ; respecter la casse.

### Pare-feu Windows

Le transfert TFTP utilise le port **UDP 69**. Si le transfert échoue sans erreur visible :
* Désactiver temporairement le pare-feu Windows, ou
* Créer une règle entrante autorisant le port UDP 69 pour l'interface `10.30.1.100`.

### Test de connectivité

Avant tout transfert, effectuer un **ping** entre le PC et l'équipement (`ping 10.30.1.2`) pour vérifier la couche réseau.

### Branchement par type d'équipement

| Équipement | Port à connecter au PC |
|------------|------------------------|
| **Switch 2960** | Port **Fa0/1** |
| **Routeur 2811** | Port **FastEthernet0/0 (Fa0/0)** |
| **Routeur 2901** | Port **GigabitEthernet0/0 (G0/0)** |
| **ASA 5512-X** | Port **Management 0/0** |

> **Important :** Le mode ROMMON utilise par défaut la première interface disponible. Un mauvais branchement empêchera le TFTP de fonctionner.

---

## 2. Export des fichiers (Sauvegarde depuis un appareil source)

Cette étape permet de récupérer le fichier `.bin` depuis un équipement fonctionnel.

> **Sans appareil source :** Les images IOS/ASA peuvent être téléchargées sur [Cisco.com](https://software.cisco.com) (compte et contrat de maintenance requis). Vérifier la **compatibilité** entre la version logicielle et le modèle matériel sur la matrice Cisco.

### A. Sur un Switch Cisco 2960

Connecter le câble Ethernet sur le port **Fa0/1** (recommandé).

1. **Configuration de l'IP de management :**
```cisco
enable
conf t
interface vlan 1
ip address 10.30.1.2 255.255.255.0
no shutdown
exit
show ip interface brief

```


*Test : Effectuer un ping depuis le PC vers 10.30.1.2.*
2. **Identification et Envoi du fichier :**
```cisco
dir flash:
! Repérez le fichier (ex: c2960-lanbasek9-MZ.150-2.SE4.bin)

copy flash:c2960-lanbasek9-MZ.150-2.SE4.bin tftp:

```


* **Address or name of remote host []?** `10.30.1.100`
* **Destination filename?** [Entrée pour valider]



### B. Sur un Routeur Cisco (2811 ou 2901)

**Important :** Connecter le câble Ethernet sur l'interface de management principale. C'est impératif pour la procédure ROMMON.

* **Routeur 2811 :** Utiliser le port **FastEthernet0/0 (Fa0/0)**.
* **Routeur 2901 :** Utiliser le port **GigabitEthernet0/0 (G0/0)**.

1. **Configuration de l'interface :**
```cisco
enable
conf t
! Choisir l'interface selon le modèle :
! Pour 2811 :
interface FastEthernet0/0
! Pour 2901 :
interface GigabitEthernet0/0
ip address 10.30.1.2 255.255.255.0
no shutdown
exit
show ip interface brief

```


2. **Envoi du fichier :**
La procédure est identique au switch (`dir flash:` puis `copy flash: tftp:`).

### C. Sur un Pare-feu ASA 5512-X

Connecter le câble Ethernet sur le port **Management 0/0**. Pour un appareil sorti d'usine ou à sauvegarder, récupérez les fichiers dans cet ordre d'importance :

| Priorité | Fichier | Description |
|----------|---------|-------------|
| **Obligatoire** | `asaXXX-smp-k8.bin` | Image OS ASA (ex: asa924-smp-k8.bin) |
| **Recommandé** | `asdm-XXX.bin` | Interface graphique ASDM |
| **Optionnel** | `anyconnect-win-XXX.pkg` | Client VPN AnyConnect (Windows) |
| **Optionnel** | `anyconnect-macos-XXX.pkg` | Client VPN AnyConnect (macOS) |
| **Optionnel** | `csd_XXX.pkg` | Cisco Secure Desktop |

1. **Configuration de l'interface Management :**
```cisco
enable
conf t
interface Management0/0
 ip address 10.30.1.2 255.255.255.0
 no shutdown
 exit
show ip interface brief
```

2. **Liste des fichiers et export vers TFTP :**
```cisco
dir disk0:
! Repérez les noms exacts des fichiers

copy disk0:/asa924-smp-k8.bin tftp:
! Address or name of remote host []? 10.30.1.100

copy disk0:/asdm-XXX.bin tftp:

! Packages optionnels (AnyConnect, CSD) :
copy disk0:/anyconnect-win-4.X.XXX.pkg tftp:
copy disk0:/csd_XXX.pkg tftp:
```

---

## 3. Import et Restauration (Mode ROMMON)

Cette procédure s'applique aux équipements dont la mémoire flash/disque interne est **vide** ou corrompue, ou bloqués en `rommon >`.

**Branchement Physique :** Voir le tableau en section 1. Résumé :
* **Switch 2960 :** Port **Fa0/1**.
* **Routeur 2811 :** Port **FastEthernet0/0**.
* **Routeur 2901 :** Port **GigabitEthernet0/0**.
* **ASA 5512-X :** Port **Management 0/0**.

*Note Routeur :* Si vous utilisez une carte Compact Flash (CF) externe, l'échange peut se faire routeur éteint.

### A. Switchs et Routeurs — Configuration des variables d'environnement

Sur l'invite de commande `rommon >`, entrer les variables suivantes **en majuscule**.

> **⚠️ Attention :** Des interférences lors de la frappe peuvent insérer des caractères illégaux invisibles. Si une variable n'est pas prise en compte, retapez-la soigneusement.

```bash
IP_ADDRESS=10.30.1.2
IP_SUBNET_MASK=255.255.255.0
DEFAULT_GATEWAY=10.30.1.100
TFTP_SERVER=10.30.1.100
TFTP_FILE=c2960-lanbasek9-MZ.150-2.SE4.bin

```

*(Remplacez `TFTP_FILE` par le nom exact de votre fichier IOS).*

### Lancement du téléchargement

Une fois les variables set, lancer la commande :

```bash
tftpdnld

```

Validez l'avertissement. Le transfert peut prendre **5 à 10 minutes** selon la taille du fichier.

### Redémarrage

Une fois le transfert terminé et le prompt `rommon` revenu, démarrez sur la nouvelle image :

```bash
boot flash:c2960-lanbasek9-MZ.150-2.SE4.bin

```

**Vérification post-restauration :** `show version` et `dir flash:` pour confirmer l'image et l'espace disque.

### B. Cas de l'ASA 5512-X (disque interne vide)

L'ASA utilise des **variables ROMMON différentes** des switchs/routeurs. Connecter le câble sur **Management 0/0**.

1. **Variables d'environnement ROMMON :**
```bash
ADDRESS=10.30.1.2
NETMASK=255.255.255.0
SERVER=10.30.1.100
GATEWAY=10.30.1.100
IMAGE=asa924-smp-k8.bin
tftp
```

> **⚠️ Attention :** Le fichier est chargé en RAM et l'ASA boote. Ce n'est **pas permanent** — il faut ensuite installer l'OS sur le disque interne (voir section 4).

**Vérification post-restauration ASA :** `show version` et `dir disk0:` pour confirmer les fichiers installés.

### Alternative SCP (Switch, Routeur, ASA)

Sur les modèles supportant SCP, cette méthode est souvent plus fiable que TFTP. Nécessite un serveur SCP (ex: OpenSSH) sur le PC et la commande `copy scp:` côté équipement. La configuration réseau reste identique.

---

## 4. Finalisation ASA 5512-X — Installation sur disque et packages

Une fois l'ASA booté via TFTP, suivez ces étapes pour le rendre **opérationnel** et autonome. *Durée indicative : 15 à 30 minutes selon les packages.*

### Étape 1 : Copier l'OS sur le disque interne

```cisco
copy tftp: disk0:
! Address or name of remote host []? 10.30.1.100
! Source filename []? asa924-smp-k8.bin
! Destination filename [asa924-smp-k8.bin]? [Entrée]
```

*Transfert : environ 5 à 10 minutes selon la taille du fichier.*

### Étape 2 : Rendre le boot permanent

```cisco
conf t
boot system disk0:/asa924-smp-k8.bin
write memory
```

### Étape 3 : Configuration de base Management

À faire **avant** ASDM si l'accès se fait via le port Management.

```cisco
conf t
interface Management0/0
 nameif management
 security-level 100
 ip address 10.30.1.2 255.255.255.0
 no shutdown
 exit
write memory
```

### Étape 4 : Installer ASDM (interface graphique)

```cisco
copy tftp: disk0:
! Charger le fichier asdm-XXX.bin (IP: 10.30.1.100)

conf t
asdm image disk0:/asdm-XXX.bin
http server enable
http 0.0.0.0 0.0.0.0 management
write memory
```

> Accès ASDM : `https://10.30.1.2` depuis le PC (même sous-réseau).

### Étape 5 : Installer les packages (AnyConnect, CSD)

Pour que l'ASA soit pleinement opérationnel (VPN, accès distant), installez les packages. *Les interfaces `inside` et `outside` doivent exister et être configurées ; sur un ASA vierge, les créer avant.*

```cisco
! Copier les fichiers .pkg depuis le TFTP
copy tftp: disk0:
! anyconnect-win-4.X.XXX.pkg
! csd_XXX.pkg (Cisco Secure Desktop)

conf t
! Activer AnyConnect (interfaces inside/outside doivent exister)
webvpn
  enable outside
  enable inside
  anyconnect image disk0:/anyconnect-win-4.X.XXX.pkg
  anyconnect enable
  tunnel-group-list enable
  exit

! Activer Cisco Secure Desktop (optionnel)
csd image disk0:/csd_XXX.pkg

write memory
```

### Étape 6 : Vérifier les licences

Les licences sont liées au numéro de série. Vérifiez leur état :

```cisco
show version
```

Contrôlez notamment :
- **Security Plus** (si applicable)
- **AnyConnect** (nombre de sessions VPN)
- **FirePOWER** (module IPS/IDS, si présent)

**Clé d'activation (ASA neuf ou réinitialisé) :**
```cisco
activation-key xxxx-xxxx-xxxx-xxxx-xxxx
```

### Étape 7 : Module FirePOWER (optionnel)

Si l'ASA 5512-X est équipé du module FirePOWER (IPS/IDS), l'installation du package SFR et la configuration de base se font via ASDM ou la CLI. Consulter la documentation Cisco FirePOWER pour l'ASA.

---

## 5. Dépannage et Réinitialisation (Password Recovery)

### Erreurs courantes TFTP / ROMMON

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| Transfert bloqué ou timeout | Pare-feu Windows, mauvais port | Ouvrir UDP 69, vérifier le répertoire TFTPD64 |
| Variables non prises en compte | Caractères illégaux, faute de frappe | Retaper soigneusement en majuscules |
| Fichier non trouvé | Mauvais nom, casse incorrecte | Vérifier le nom exact avec `dir flash:` ou `dir disk0:` |
| Pas de réponse réseau | Mauvais branchement, mauvaise interface | Vérifier le tableau de branchement (section 1), tester le ping |

### Problème d'affichage (Caractères bizarres)

Si le texte est illisible au démarrage, testez les vitesses de connexion (Baud Rate) dans Putty dans cet ordre :
`9600` (défaut), `19200`, `38400`, `57600`, `115200`.

### Procédure de Reset Mot de Passe

| Matériel | Action au boot | Commande clé |
|----------|----------------|--------------|
| **Routeur** | `CTRL+BREAK` (Putty) ou `ALT+B` (TeraTerm) dans les 5 premières secondes | `confreg 0x2142` puis `reset` |
| **Switch** | Maintenir bouton **MODE** (débrancher alim, maintenir, rebrancher) | `flash_init` puis `rename flash:config.text flash:config.old` |
| **ASA** | Appuyer sur **ESCAPE** au démarrage | `confreg 0x41` (puis remettre `0x1` après le reset) |

#### Pour Routeur (ex: 2811)

1. **Entrer en ROMMON :** Redémarrer le routeur et faire `CTRL+Break` (Putty) ou `ALT+B` (TeraTerm) dans les 5 premières secondes.
2. **Contourner la conf :**
```bash
confreg 0x2142
reset

```


3. **Restaurer après boot :**
```cisco
enable
copy startup-config running-config
conf t
enable secret <nouveau_mdp>
config-register 0x2102
end
write memory

```



#### Pour Switch (ex: 2960)

1. **Entrer en Bootloader :** Débrancher l'alim. Maintenir le bouton **MODE** en façade. Rebrancher l'alim. Relâcher le bouton quand la LED SYST cesse de clignoter.
2. **Initialiser et renommer :**
```bash
flash_init
rename flash:config.text flash:config.old
boot

```


3. **Restaurer après boot :**
Répondre `no` au setup initial.
```cisco
enable
rename flash:config.old flash:config.text
copy flash:config.text running-config
conf t
enable secret <nouveau_mdp>
end
write memory
```

#### Pour ASA 5512-X

1. **Entrer en ROMMON :** Redémarrer l'ASA et appuyer sur **ESCAPE** dans les premières secondes du boot.
2. **Contourner la configuration :**
```bash
confreg 0x41
reset
```
3. **Restaurer après boot :** Répondre `no` au setup initial.
```cisco
enable
conf t
config-register 0x1
enable password <nouveau_mdp>
enable secret <nouveau_mdp>
end
write memory
```

---

## 6. Synthèse rapide — Checklist technicien

| Étape | Switch / Routeur | ASA 5512-X |
|-------|------------------|------------|
| **1. Préparer** | PC : 10.30.1.100, TFTPD64, Putty | Idem |
| **2. Export** | `copy flash: tftp:` | `copy disk0: tftp:` (OS + ASDM + .pkg) |
| **3. Branchement** | Fa0/1 (2960), Fa0/0 (2811), G0/0 (2901) | Management 0/0 |
| **4. ROMMON** | `IP_ADDRESS`, `TFTP_SERVER`, `TFTP_FILE`, `tftpdnld` | `ADDRESS`, `SERVER`, `IMAGE`, `tftp` |
| **5. Boot** | `boot flash:xxx.bin` | Boot automatique (puis installer sur disk0) |
| **6. Finalisation** | — | Copier OS → disk0, boot system, config Management, ASDM, packages AnyConnect/CSD |
