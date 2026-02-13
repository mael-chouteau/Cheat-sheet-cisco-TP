# Procédure de Sauvegarde et Restauration d'IOS Cisco (Switch 2960 & Routeur 2811/2901)

Ce document détaille la procédure pour extraire une image IOS d'un équipement fonctionnel et la restaurer sur un équipement vierge ou bloqué en mode ROMMON via TFTP.

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

---

## 2. Extraction de l'IOS (Sauvegarde)

Cette étape permet de récupérer le fichier `.bin` depuis un équipement fonctionnel.

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

---

## 3. Restauration d'IOS (Mode ROMMON)

Cette procédure s'applique aux équipements sans IOS valide ou bloqués en `rommon >`.

**Branchement Physique :**

* **Switch 2960 :** Port **Fa0/1**. (Le mode ROMMON utilise par défaut la première interface disponible. Si vous utilisez un autre port (ex: Fa0/2 ou G0/1), le tftp ne fonctionnera pas.)
* **Routeur 2811 :** Port **FastEthernet0/0**.
* **Routeur 2901 :** Port **GigabitEthernet0/0**.(Le ROMMON utilise cette interface par défaut).
* *Note Routeur :* Si vous utilisez une carte Compact Flash (CF) externe, l'échange peut se faire routeur éteint.

### Configuration des variables d'environnement

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

Validez l'avertissement. Le transfert peut prendre jusqu'à **10 minutes**.

### Redémarrage

Une fois le transfert terminé et le prompt `rommon` revenu, démarrez sur la nouvelle image :

```bash
boot flash:c2960-lanbasek9-MZ.150-2.SE4.bin

```

---

## 4. Dépannage et Réinitialisation (Password Recovery)

### Problème d'affichage (Caractères bizarres)

Si le texte est illisible au démarrage, testez les vitesses de connexion (Baud Rate) dans Putty dans cet ordre :
`9600` (défaut), `19200`, `38400`, `57600`, `115200`.

### Procédure de Reset Mot de Passe

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
