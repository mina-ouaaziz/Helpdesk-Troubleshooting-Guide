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
Symptôme : L'ordinateur ne s'allume pas ou reste bloqué avant le chargement de Windows.
Causes fréquentes :

Câble d'alimentation débranché ou bloc d'alimentation défaillant
RAM mal insérée ou défectueuse
Disque dur défaillant
Corruption du secteur de démarrage (MBR/GPT)

Étapes de résolution :

Vérifier que le câble d'alimentation est bien branché (côté PC et côté prise)
Tester la prise avec un autre appareil pour s'assurer qu'elle est bien alimentée
Débrancher tous les périphériques USB et relancer
Si le PC s'allume mais ne démarre pas Windows : booter sur un support USB Windows → Réparer l'ordinateur → Réparation du démarrage
En ligne de commande (WinRE) :

cmdbootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd

⚠️ Note UEFI/GPT : Sur les machines récentes, bootrec /fixboot peut renvoyer une erreur "Accès refusé". Dans ce cas, tenter uniquement bootrec /rebuildbcd, ou vérifier que le volume EFI est bien monté avant de relancer la commande.

Escalade N2 si : Disque dur non détecté au BIOS, erreurs matérielles répétées.

2. Mot de passe oublié / compte AD verrouillé
Symptôme : L'utilisateur ne peut plus se connecter à sa session Windows ou à une application métier.
Causes fréquentes :

Mot de passe expiré
Compte verrouillé après trop de tentatives échouées
Touche Majuscule activée lors de la saisie (ça arrive plus souvent qu'on ne le croit)

Étapes de résolution (côté admin AD) :

Ouvrir Utilisateurs et ordinateurs Active Directory
Rechercher le compte utilisateur
Clic droit → Propriétés → onglet Compte
Décocher "Le compte est verrouillé" si applicable
Cocher "L'utilisateur doit changer le mot de passe à la prochaine ouverture de session"
Clic droit → Réinitialiser le mot de passe → fournir un mot de passe temporaire sécurisé

Via PowerShell :
powershell# Déverrouiller un compte
Unlock-ADAccount -Identity "prenom.nom"

# Réinitialiser le mot de passe
# ⚠️ Remplacer par un mot de passe temporaire conforme à la politique de sécurité de l'entreprise
# Ne jamais utiliser de vrais mots de passe dans un script versionné
Set-ADAccountPassword -Identity "prenom.nom" -Reset -NewPassword (ConvertTo-SecureString "MotDePasseTmp_A_Changer!" -AsPlainText -Force)

# Forcer le changement au prochain login
Set-ADUser -Identity "prenom.nom" -ChangePasswordAtLogon $true
Escalade N2 si : Verrouillages répétés sans explication, suspicion de compromission de compte.

3. Imprimante non reconnue ou ne répond pas
Symptôme : L'imprimante n'apparaît pas dans Windows ou les impressions restent bloquées dans la file.
Causes fréquentes :

Spouleur d'impression planté
Driver corrompu ou absent
Imprimante hors ligne ou déconnectée du réseau

Étapes de résolution :

Vérifier que l'imprimante est allumée et bien connectée (USB ou réseau)
Vider la file d'impression et redémarrer le spouleur — lancer ces commandes dans une invite de commande administrateur :

cmdnet stop spooler
del /Q /F /S "%systemroot%\System32\spool\PRINTERS\*.*"
net start spooler

Vérifier dans Périphériques et imprimantes que l'imprimante n'est pas en mode "Hors ligne"
Désinstaller et réinstaller le driver depuis le site du constructeur
Pour une imprimante réseau : vérifier que son IP est joignable via ping [IP_imprimante]

Escalade N2 si : Problème persistant sur toutes les imprimantes du réseau, erreur de driver impossible à résoudre.

4. Connexion VPN impossible
Symptôme : L'utilisateur ne parvient pas à établir la connexion VPN depuis son domicile.
Causes fréquentes :

Mauvais identifiants ou certificat expiré
Port VPN bloqué par le pare-feu local ou le FAI
Client VPN corrompu
Adresse du serveur VPN incorrecte

Étapes de résolution :

Vérifier que l'utilisateur a bien accès à Internet sans VPN
Vérifier les identifiants (login, mot de passe, domaine)
S'assurer que le client VPN est à jour
Désactiver temporairement le pare-feu Windows et retenter
Vérifier l'adresse du serveur VPN dans la configuration du client
Tenter depuis un autre réseau (partage de connexion mobile) — si ça fonctionne, le problème vient du FAI ou du réseau local
Réinstaller le client VPN si nécessaire

Escalade N2 si : Le serveur VPN est inaccessible depuis plusieurs utilisateurs en même temps (incident global), ou problème de certificat côté serveur.

5. Profil Windows corrompu
Symptôme : À la connexion, message "Vous avez été connecté avec un profil temporaire" ou bureau vide avec applications manquantes.
Causes fréquentes :

Arrêt brutal du PC pendant une mise à jour
Profil utilisateur corrompu dans la base de registre
Disque dur avec secteurs défectueux

Étapes de résolution :

Redémarrer le PC et retenter la connexion (parfois ça suffit)
Se connecter avec un compte administrateur local
Ouvrir regedit → naviguer vers :

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList

Identifier la clé correspondant au profil corrompu (SID de l'utilisateur)
Si deux clés existent — l'une avec .bak, l'autre sans — voici la procédure dans l'ordre :

Renommer la clé sans .bak en y ajoutant .old (ex: S-1-5-21-xxxx → S-1-5-21-xxxx.old)
Renommer la clé avec .bak en supprimant le .bak (ex: S-1-5-21-xxxx.bak → S-1-5-21-xxxx)


Redémarrer et retenter la connexion avec le compte utilisateur
Si le profil est irrécupérable : créer un nouveau profil et migrer les données depuis C:\Users\ancienProfil

Escalade N2 si : Corruption matérielle suspectée — lancer chkdsk /f /r au préalable pour vérifier l'état du disque.

6. Pas d'accès Internet
Symptôme : Le navigateur affiche "Pas d'accès Internet" ou "DNS_PROBE_FINISHED".
Causes fréquentes :

Carte réseau désactivée
Configuration IP incorrecte (IP fixe mal renseignée)
Problème DNS
Câble réseau débranché ou défaillant

Étapes de résolution :

Vérifier l'icône réseau dans la barre des tâches
ipconfig /all → vérifier qu'une IP cohérente est attribuée
Tenter un ping 8.8.8.8 (DNS Google) → si ça répond, la connectivité réseau est OK mais le DNS est en cause
Si problème DNS :

cmdipconfig /flushdns
netsh winsock reset

⚠️ netsh winsock reset nécessite un redémarrage pour prendre effet.


Si le ping ne répond pas : ipconfig /release puis ipconfig /renew
Désactiver puis réactiver la carte réseau dans le Gestionnaire de périphériques
Tester avec un autre câble ou en Wi-Fi pour éliminer une cause matérielle

Escalade N2 si : Plusieurs postes touchés simultanément (problème infrastructure), ou si le serveur DHCP ne répond plus.

7. Écran bleu (BSOD)
Symptôme : Écran bleu avec un code d'erreur, redémarrage automatique du PC.
Causes fréquentes :

Driver incompatible ou corrompu (souvent après une mise à jour)
RAM défectueuse
Surchauffe du matériel
Infection malware

Étapes de résolution :

Noter le code d'erreur affiché (ex: IRQL_NOT_LESS_OR_EQUAL, PAGE_FAULT_IN_NONPAGED_AREA)
Analyser le fichier de dump dans C:\Windows\Minidump avec WinDbg ou BlueScreenView (outil tiers NirSoft, gratuit)
Identifier le driver incriminé et le mettre à jour ou le désinstaller
Vérifier la RAM avec l'outil intégré Windows :

cmdmdsched.exe

Vérifier les températures avec HWMonitor ou Core Temp
Si le BSOD est apparu après une mise à jour récente : restaurer le système à un point de restauration antérieur

Escalade N2 si : BSOD répétés sans cause identifiable, suspicion de panne matérielle.

8. Messagerie Outlook ne se synchronise plus
Symptôme : Les emails ne se chargent plus, Outlook affiche "Déconnecté" ou "Tentative de connexion".
Causes fréquentes :

Fichier OST/PST corrompu
Mot de passe expiré
Profil Outlook corrompu
Problème de connectivité Exchange

Étapes de résolution :

Vérifier la connexion Internet
Tester si le mot de passe fonctionne sur OWA (Outlook Web App) — si ça bloque là aussi, c'est le mot de passe
Réparer le fichier de données Outlook :

Fichier → Paramètres du compte → Fichiers de données → Ouvrir l'emplacement du fichier
Lancer SCANPST.EXE sur le fichier .ost ou .pst


Si ça ne suffit pas, supprimer et recréer le profil Outlook :

Panneau de configuration → Courrier → Afficher les profils → Ajouter


Vérifier l'état du service Exchange avec l'équipe N2

Escalade N2 si : Problème global touchant plusieurs utilisateurs, erreur côté serveur Exchange/O365.

9. Partage réseau inaccessible
Symptôme : L'utilisateur ne peut pas accéder à un lecteur réseau mappé ou à un dossier partagé.
Causes fréquentes :

Session expirée ou mot de passe changé récemment
Lecteur réseau non remappé après un redémarrage
Problème de droits NTFS ou de partage
Serveur de fichiers hors ligne

Étapes de résolution :

Tester l'accès direct via \\serveur\partage dans l'explorateur de fichiers
Vérifier que le serveur répond : ping [nom_serveur]
Vérifier (et mettre à jour si besoin) les identifiants dans le Gestionnaire d'informations d'identification Windows
Remapper le lecteur manuellement :

cmdnet use Z: \\serveur\partage /persistent:yes

Vérifier les droits avec un compte admin sur le dossier partagé

Escalade N2 si : Serveur de fichiers inaccessible pour tous les utilisateurs, problème de droits complexe à démêler.

10. PC lent / performances dégradées
Symptôme : Le PC met du temps à démarrer, les applications rament, le système est globalement lent.
Causes fréquentes :

Disque dur presque plein
Trop de programmes au démarrage
Malware ou processus parasite
RAM insuffisante
Disque dur mécanique défaillant (HDD)

Étapes de résolution :

Vérifier l'espace disque disponible — en dessous de 10% libre, les performances chutent
Lancer le nettoyage de disque : cleanmgr
Désactiver les programmes inutiles au démarrage :

Gestionnaire des tâches → onglet Démarrage


Repérer les processus gourmands dans le Gestionnaire des tâches (CPU/RAM)
Lancer une analyse antivirus complète
Vérifier l'état du disque :

cmdchkdsk C: /f /r
Ou via PowerShell (recommandé sur Windows 10/11) :
powershellGet-PhysicalDisk | Select FriendlyName, HealthStatus

Vérifier les températures — une surchauffe entraîne du throttling et ralentit tout

Escalade N2 si : Disque dur en état critique, remplacement de RAM nécessaire.

📌 Bonnes pratiques générales

Commencer par le plus simple : câble, redémarrage, droits utilisateur — ça résout 80% des tickets
Documenter chaque intervention dans l'outil ITSM (GLPI, ServiceNow…)
Ne jamais toucher aux données d'un utilisateur sans son accord
Escalader sans hésiter si après 20 minutes on n'avance plus — ce n'est pas un aveu d'échec, c'est de la bonne gestion d'incident

---

*Rédigé par Mina Ouaaziz — Technicienne Systèmes & Réseaux*  
*Compétences : Windows Server · Active Directory · pfSense · Proxmox · GLPI · Zabbix · PowerShell
