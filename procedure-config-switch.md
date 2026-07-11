# 📑 Procédure de Déploiement : Switch Aruba 2530

> [!NOTE]
> **Référence matériel :** Aruba Networks (HPE)  
> **Objectif :** Configuration complète, VLANs, IP de management et recette réseau.

---
## Préparation

Avant de commencer la configuration du switch Aruba, les éléments suivants doivent être disponibles, installés et fonctionnels :

###  Matériel nécessaire

- Adaptateur **USB‑Série** compatible (UC232A, FTDI, Prolific, etc.)
- Câble **RJ45 Console** Aruba / HP
- PC portable avec droits administrateur
- Accès physique au switch

---

### Pilote pour l’adaptateur USB‑Série (Port COM)

Un pilote est nécessaire pour que Windows reconnaisse l’adaptateur USB‑Série utilisé pour accéder à la console du switch.

📦 **Pilote UC232A (Windows)**  
> *Fichier d’installation :* `UC232A_Windows_Setup.exe`  
> *(À placer dans le dossier du projet ou dans un espace partagé interne)*

> [!NOTE]  
> Si le port COM n’apparaît pas dans le Gestionnaire de périphériques, le pilote n’est pas installé correctement.

---

### Logiciel de terminal : PuTTY

PuTTY est utilisé pour accéder à la console série du switch.

🔗 **Téléchargement officiel :**  
https://putty.org/index.html

> [!TIP]
> **Paramètres PuTTY recommandés pour la connexion console :**
>
> ```bash
> Serial line : COMX   (ex : COM3)
> Speed       : 9600
> Data bits   : 8
> Stop bits   : 1
> Parity      : None
> Flow control: None
> ```
![Configuration PuTTY](images/config_putty.png)

## Ⅰ. Réinitialisation Totale (Monitor ROM)

À utiliser pour effacer toute configuration résiduelle et repartir sur une base saine.

### 🔧 Accès au boot
- Redémarrer électriquement le switch
- Appuyer sur `0` de manière répétée

### 🧹 Effacement
```bash
erase-all
```
Confirmer avec `y`

### 🔄 Redémarrage
```bash
boot
```
Appuyer deux fois sur **Entrée** au message *Waiting for Speed Sense*


---

## Ⅱ. Initialisation et Sécurité

Configuration de l’identité du matériel et restriction des accès.

```bash
conf t
hostname LVS_197
password manager
# Saisir le mot de passe deux fois
exit
```
![Nommage du switch](images/nommage_switch.png)
> [!IMPORTANT]
> 🔐 Le mot de passe **manager** protège l’accès privilégié (#).  
> *Il doit être stocké dans le coffre-fort de mots de passe.*
 
---

## Ⅲ. Architecture des VLANs et Ports

Segmentation des flux : Data, Voix et Wi-Fi.

### 1. Création des VLANs

> [!NOTE]
> Cette migration a aussi servi à harmoniser le plan de VLANs avec le reste du parc de la mairie : l'ancien VLAN 44 (TOIP) est fusionné dans le VLAN 2 (VoIP), et l'ancien VLAN 207 (WiFi LVS) devient le VLAN 210 (bornes WiFi). Les VLANs 4, 150 et 151 sont créés en réserve pour des besoins futurs, sans port assigné pour l'instant.

```bash
conf t
vlan 2    name "VoIP"        exit
vlan 210  name "WIFILVS"     exit
vlan 208  name "management"  exit
vlan 4    name "RESERVE_4"   exit
vlan 150  name "RESERVE_150" exit
vlan 151  name "RESERVE_151" exit
```
![Configuration des VLANs](images/show-vlan.png)

### 2. Assignation des ports

#### VLAN 1 (Accès / natif)

```bash
vlan 1
   no untagged 1,3,19
   untagged 2,4-18,20-21,23-26
   exit
```

#### VLAN voix & data

```bash
vlan 2
   tagged 1-26
   exit
```

#### Uplink & services

```bash
vlan 210
   tagged 25
   exit

vlan 208
   tagged 25
   exit
```

> [!TIP]
> **Pour vérifier quel VLAN est affecté à quel port :** `show vlan [ID_DU_VLAN]`
>
> **Exemple avec le VLAN 208 :**
>
> ![Vérification VLAN 208](images/show_vlan208.png)

---

## Ⅳ. Interface de Management IP
*Configuration de l'accès distant via le VLAN d'administration.*

```bash
conf t
management-vlan 208
vlan 208
   ip address 192.168.198.197 255.255.255.0
   exit

# Définition de la passerelle (IP du Cœur de Réseau)
ip default-gateway 192.168.198.1
```

---

## Ⅴ. Procédure de Recette (Test de Ping)

Validation avant mise en production.

### 1. Isolation du port test

```bash
conf t
vlan 208
   untagged 10
   exit
```

### 2. Test

- PC branché sur port 10  
- IP : `192.168.198.100`  
- Commande :

```bash
ping 192.168.198.197
```

> [!TIP]
> 📸 **PHOTO 3 : Ping réussi (0% loss)**

### 3. Remise en état

```bash
conf t
vlan 1
   untagged 10
   exit

vlan 208
   no untagged 10
   exit
```

---

## Ⅵ. Sauvegarde et Clôture

```bash
write memory
```

> [!CAUTION]
> ⚠️ Sans validation **[OK]**, la configuration sera perdue au redémarrage.

---

## Ⅶ. Déploiement Physique

- Installation en baie (rack)
- Branchement uplink sur port 25
- Vérification d’accès web :

```
http://192.168.198.197
```

> [!TIP]
> 📸 **PHOTO 4 : Interface web du switch**
