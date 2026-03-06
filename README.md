# 🛠️ Helpdesk Troubleshooting Guide

Guide de résolution des incidents les plus courants en support technique N1/N2.  
Chaque fiche suit la structure : **Symptôme → Causes fréquentes → Étapes de résolution → Escalade N2.**

---

## 📋 Sommaire

1. [PC ne démarre pas](#1-pc-ne-démarre-pas)
2. [Mot de passe oublié / compte AD verrouillé](#2-mot-de-passe-oublié--compte-ad-verrouillé)
3. [Imprimante non reconnue ou ne répond pas](#3-imprimante-non-reconnue-ou-ne-répond-pas)
4. [Connexion VPN impossible](#4-connexion-vpn-impossible)
5. [Profil Windows corrompu](#5-profil-windows-corrompu)
6. [Pas d'accès Internet](#6-pas-daccès-internet)
7. [Écran bleu (BSOD)](#7-écran-bleu-bsod)
8. [Messagerie Outlook ne se synchronise plus](#8-messagerie-outlook-ne-se-synchronise-plus)
9. [Partage réseau inaccessible](#9-partage-réseau-inaccessible)
10. [PC lent / performances dégradées](#10-pc-lent--performances-dégradées)

---

## 1. PC ne démarre pas

**Symptôme :** L'ordinateur ne s'allume pas ou reste bloqué avant le chargement de Windows.

**Causes fréquentes :**
- Câble d'alimentation débranché ou bloc d'alimentation défaillant
- RAM mal insérée ou défectueuse
- Disque dur défaillant
- Corruption du secteur de démarrage (MBR/GPT)

**Étapes de résolution :**
1. Vérifier que le câble d'alimentation est bien branché (PC et prise murale)
2. Vérifier que la prise est alimentée (tester avec un autre appareil)
3. Retirer les périphériques USB et relancer
4. Si le PC s'allume mais ne démarre pas Windows : booter sur un support USB Windows → `Réparer l'ordinateur` → `Réparation du démarrage`
5. En ligne de commande (WinRE) :
```
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
```

**Escalade N2 si :** Disque dur non détecté au BIOS, erreurs matérielles répétées.

---

## 2. Mot de passe oublié / compte AD verrouillé

**Symptôme :** L'utilisateur ne peut plus se connecter à sa session Windows ou à une application métier.

**Causes fréquentes :**
- Mot de passe expiré
- Compte verrouillé après trop de tentatives échouées
- Majuscule activée lors de la saisie

**Étapes de résolution (côté admin AD) :**
1. Ouvrir **Utilisateurs et ordinateurs Active Directory**
2. Rechercher le compte utilisateur
3. Clic droit → **Propriétés** → onglet **Compte**
4. Décocher "Le compte est verrouillé" si applicable
5. Cocher "L'utilisateur doit changer le mot de passe à la prochaine ouverture de session"
6. Clic droit → **Réinitialiser le mot de passe** → fournir un mot de passe temporaire sécurisé

**Via PowerShell :**
```powershell
# Déverrouiller un compte
Unlock-ADAccount -Identity "prenom.nom"

# Réinitialiser le mot de passe
Set-ADAccountPassword -Identity "prenom.nom" -Reset -NewPassword (ConvertTo-SecureString "MotDePasse!Tmp1" -AsPlainText -Force)

# Forcer le changement au prochain login
Set-ADUser -Identity "prenom.nom" -ChangePasswordAtLogon $true
```

**Escalade N2 si :** Verrouillages répétés sans explication, suspicion de compromission de compte.

---

## 3. Imprimante non reconnue ou ne répond pas

**Symptôme :** L'imprimante n'apparaît pas dans Windows ou les impressions restent bloquées dans la file.

**Causes fréquentes :**
- Spouleur d'impression planté
- Driver corrompu ou absent
- Imprimante hors ligne ou déconnectée du réseau

**Étapes de résolution :**
1. Vérifier que l'imprimante est allumée et connectée (USB ou réseau)
2. Vider la file d'impression et redémarrer le spouleur :
```cmd
net stop spooler
del /Q /F /S "%systemroot%\System32\spool\PRINTERS\*.*"
net start spooler
```
3. Vérifier dans `Périphériques et imprimantes` que l'imprimante n'est pas en mode "Hors ligne"
4. Désinstaller et réinstaller le driver depuis le site du constructeur
5. Pour une imprimante réseau : vérifier que son IP est joignable via `ping [IP_imprimante]`

**Escalade N2 si :** Problème persistant sur toutes les imprimantes du réseau, erreur de driver impossible à résoudre.

---

## 4. Connexion VPN impossible

**Symptôme :** L'utilisateur ne parvient pas à établir la connexion VPN depuis son domicile.

**Causes fréquentes :**
- Mauvaises identifiants ou certificat expiré
- Port VPN bloqué par le pare-feu local ou FAI
- Client VPN corrompu
- Adresse du serveur VPN incorrecte

**Étapes de résolution :**
1. Vérifier que l'utilisateur a accès à Internet sans VPN
2. Vérifier les identifiants (login, mot de passe, domaine)
3. Vérifier que le client VPN est à jour
4. Désactiver temporairement le pare-feu Windows et retenter
5. Vérifier l'adresse du serveur VPN dans la configuration du client
6. Tenter depuis un autre réseau (partage de connexion mobile) pour isoler un blocage FAI
7. Réinstaller le client VPN si nécessaire

**Escalade N2 si :** Le serveur VPN est inaccessible depuis plusieurs utilisateurs simultanément (incident global), ou problème de certificat côté serveur.

---

## 5. Profil Windows corrompu

**Symptôme :** À la connexion, message "Vous avez été connecté avec un profil temporaire" ou bureau vide avec applications manquantes.

**Causes fréquentes :**
- Arrêt brutal du PC lors d'une mise à jour
- Profil utilisateur corrompu dans la base de registre
- Disque dur avec secteurs défectueux

**Étapes de résolution :**
1. Redémarrer le PC et retenter la connexion (peut suffire)
2. Se connecter avec un compte administrateur local
3. Ouvrir `regedit` → naviguer vers :
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList
```
4. Identifier la clé correspondant au profil (SID de l'utilisateur)
5. Si une clé avec `.bak` est présente : renommer la clé sans `.bak` en ajoutant `.old`, puis supprimer le `.bak` de l'autre
6. Redémarrer et retenter la connexion
7. Si le profil est irrécupérable : créer un nouveau profil et migrer les données depuis `C:\Users\ancienProfil`

**Escalade N2 si :** Corruption matérielle suspectée (lancer `chkdsk /f /r` au préalable).

---

## 6. Pas d'accès Internet

**Symptôme :** Le navigateur affiche "Pas d'accès Internet" ou "DNS_PROBE_FINISHED".

**Causes fréquentes :**
- Carte réseau désactivée
- Configuration IP incorrecte (IP fixe mal renseignée)
- Problème DNS
- Câble réseau débranché ou défaillant

**Étapes de résolution :**
1. Vérifier l'icône réseau dans la barre des tâches
2. `ipconfig /all` → vérifier qu'une IP cohérente est attribuée
3. Tenter un `ping 8.8.8.8` (Google DNS) → si ça répond, le problème est DNS
4. Si problème DNS :
```cmd
ipconfig /flushdns
netsh winsock reset
```
5. Si pas de réponse au ping : `ipconfig /release` puis `ipconfig /renew`
6. Désactiver/réactiver la carte réseau dans le Gestionnaire de périphériques
7. Tester avec un autre câble ou en Wi-Fi pour isoler le problème matériel

**Escalade N2 si :** Plusieurs postes touchés simultanément (problème infrastructure), ou si le serveur DHCP ne répond plus.

---

## 7. Écran bleu (BSOD)

**Symptôme :** Écran bleu avec un code d'erreur, redémarrage automatique du PC.

**Causes fréquentes :**
- Driver incompatible ou corrompu (souvent après une mise à jour)
- RAM défectueuse
- Surchauffe du matériel
- Infection malware

**Étapes de résolution :**
1. Noter le code d'erreur affiché (ex: `IRQL_NOT_LESS_OR_EQUAL`, `PAGE_FAULT_IN_NONPAGED_AREA`)
2. Analyser le fichier de dump : `C:\Windows\Minidump` avec **WinDbg** ou **BlueScreenView**
3. Identifier le driver incriminé et le mettre à jour ou le désinstaller
4. Vérifier la RAM avec **Windows Memory Diagnostic** :
```cmd
mdsched.exe
```
5. Vérifier les températures avec **HWMonitor** ou **Core Temp**
6. Si récent après mise à jour : restaurer le système à un point antérieur

**Escalade N2 si :** BSOD répétés sans cause identifiable, suspicion de panne matérielle.

---

## 8. Messagerie Outlook ne se synchronise plus

**Symptôme :** Les emails ne se chargent plus, Outlook affiche "Déconnecté" ou "Tentative de connexion".

**Causes fréquentes :**
- Fichier OST/PST corrompu
- Mot de passe expiré
- Profil Outlook corrompu
- Problème de connectivité Exchange

**Étapes de résolution :**
1. Vérifier la connexion Internet
2. Vérifier que le mot de passe n'est pas expiré (tester sur OWA)
3. Réparer le fichier de données Outlook :
   - `Fichier` → `Paramètres du compte` → `Fichiers de données` → `Ouvrir l'emplacement du fichier`
   - Lancer `SCANPST.EXE` sur le fichier .ost/.pst
4. Supprimer et recréer le profil Outlook :
   - `Panneau de configuration` → `Courrier` → `Afficher les profils` → `Ajouter`
5. Vérifier l'état du service Exchange avec l'équipe N2

**Escalade N2 si :** Problème global sur plusieurs utilisateurs, erreur côté serveur Exchange/O365.

---

## 9. Partage réseau inaccessible

**Symptôme :** L'utilisateur ne peut pas accéder à un lecteur réseau mappé ou à un dossier partagé.

**Causes fréquentes :**
- Session expirée ou mot de passe changé
- Lecteur réseau non remappé après redémarrage
- Problème de droits NTFS ou de partage
- Serveur de fichiers hors ligne

**Étapes de résolution :**
1. Tester l'accès direct via `\\serveur\partage` dans l'explorateur
2. Vérifier que le serveur est joignable : `ping [nom_serveur]`
3. Vérifier les identifiants dans le gestionnaire d'informations d'identification Windows
4. Remapper le lecteur manuellement :
```cmd
net use Z: \\serveur\partage /persistent:yes
```
5. Vérifier les droits avec un compte admin sur le dossier partagé

**Escalade N2 si :** Serveur de fichiers inaccessible pour tous les utilisateurs, problème de droits complexe.

---

## 10. PC lent / performances dégradées

**Symptôme :** Le PC met du temps à démarrer, les applications rament, le système est globalement lent.

**Causes fréquentes :**
- Disque dur presque plein
- Trop de programmes au démarrage
- Malware ou processus parasite
- RAM insuffisante
- Disque dur mécanique défaillant (HDD)

**Étapes de résolution :**
1. Vérifier l'espace disque disponible (minimum 10% libre recommandé)
2. Lancer le nettoyage de disque : `cleanmgr`
3. Désactiver les programmes inutiles au démarrage :
   - `Gestionnaire des tâches` → onglet `Démarrage`
4. Vérifier les processus gourmands dans le Gestionnaire des tâches (CPU/RAM)
5. Lancer une analyse antivirus complète
6. Vérifier l'état du disque :
```cmd
wmic diskdrive get status
chkdsk C: /f /r
```
7. Vérifier les températures (surchauffe = throttling)

**Escalade N2 si :** Disque dur en état critique, remplacement de RAM nécessaire.

---

## 📌 Bonnes pratiques générales

- **Toujours commencer par le plus simple** : câble, redémarrage, droits utilisateur
- **Documenter chaque intervention** dans l'outil ITSM (GLPI, ServiceNow…)
- **Ne jamais intervenir sans accord de l'utilisateur** sur ses données
- **Escalader sans hésiter** si la résolution dépasse 20 minutes sans avancée

---

*Rédigé par Mina Ouaaziz — Technicienne Systèmes & Réseaux*  
*Compétences : Windows Server, Active Directory, pfSense, Proxmox, GLPI, Zabbix*
