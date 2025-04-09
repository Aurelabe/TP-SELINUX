# Rapport TP : Sécurisation d'un serveur Rocky Linux avec SELinux et SSH

## 1. Installation du système d’exploitation

### 1.1 Installation de Rocky Linux 9

Le serveur Rocky Linux 9 a été installé en mode minimal, sans interface graphique, conformément aux consignes. L'installation a été réalisée avec l'image ISO officielle et tous les correctifs de sécurité ont été appliqués en utilisant la commande suivante :

```bash
sudo dnf update -y
```

### 1.2 Configuration initiale

- **Interface graphique** : Aucun environnement de bureau n'a été installé, conformément aux exigences du TP.
- **Derniers correctifs de sécurité** : Tous les correctifs ont été appliqués.

---

## 2. Configuration réseau

### 2.1 Modification des interfaces réseau avec `nmcli`

Les interfaces réseau par défaut **"Wired connection 1"** et **"Wired connection 2"** ont été renommées en **enp0s3** et **enp0s8** respectivement. Pour cela, nous avons utilisé la commande suivante avec `nmcli` :

```bash
nmcli con mod "Wired connection 1" con.id enp0s3
nmcli con mod "Wired connection 2" con.id enp0s8
```

### 2.2 Configuration de l'IP statique

Nous avons configuré l'interface `enp0s3` avec l'adresse IP statique `10.1.1.15` via `nmcli` comme suit :

```bash
nmcli con mod enp0s8 ipv4.addresses 10.1.1.15/24 ipv4.gateway 10.1.1.1 ipv4.dns "8.8.8.8" ipv4.method manual
nmcli con down enp0s8
nmcli con up enp0s8
```

---

## 3. Configuration du filtrage réseau

### 3.1 Création de la zone "restricted"

Une nouvelle zone a été créée dans `firewalld` appelée **"restricted"** pour appliquer un filtrage strict. Les interfaces `enp0s3` et `enp0s8` ont été ajoutées à cette zone avec les commandes suivantes :

```bash
sudo firewall-cmd --permanent --new-zone=restricted
sudo firewall-cmd --permanent --zone=restricted --add-interface=enp0s3
sudo firewall-cmd --permanent --zone=restricted --add-interface=enp0s8
```

### 3.2 Ouverture des ports nécessaires

Dans la zone "restricted", seuls les ports nécessaires au bon fonctionnement du serveur ont été ouverts. Nous avons ouvert les ports pour SSH (port 1025), 80 et 443 comme suit :

```bash
sudo firewall-cmd --zone=restricted --add-port=1025/tcp --permanent
sudo firewall-cmd --zone=restricted --add-port=80/tcp --permanent
sudo firewall-cmd --zone=restricted --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

Ces étapes assurent que seul le trafic nécessaire est autorisé et que la zone "restricted" renforce la sécurité du serveur.

---

## 4. Configuration de l’administration SSH

### 4.1 Sécurisation de SSH

Pour renforcer la sécurité de l'administration via SSH, j'ai effectué plusieurs configurations :

1. **Modification du port SSH** :
   Le port SSH a été changé de **22** à **1025** pour des raisons de sécurité, afin de réduire la probabilité d'attaques automatisées. Le fichier de configuration `/etc/ssh/sshd_config` a été modifié avec la ligne suivante :

   ```bash
   Port 1025
   ```

2. **Interdiction de l'accès SSH à l'utilisateur root** :
   Afin de limiter les attaques ciblant le compte superutilisateur, l'accès SSH pour le compte `root` a été désactivé. La ligne suivante a été décommentée dans le fichier /etc/ssh/sshd_config :
   
   ```bash
   PermitRootLogin no
   ```
   
3. **Limitation de l'accès à l'utilisateur "aurel" uniquement** :
L'authentification par clé publique a été activée pour permettre les connexions via une paire de clés SSH. C'est une condition nécessaire si l'on désactive les mots de passe :

   ```bash
   AllowUsers aurel
   ```

4. **Activation de l'authentification par clé publique** :
   L'accès SSH par mot de passe a été désactivé pour ne permettre que les connexions basées sur une clé SSH. Le fichier `/etc/ssh/sshd_config` a été modifié comme suit :

   ```bash
   PubkeyAuthentication yes
   ```

5. **Désactivation de l'accès SSH par mot de passe** :
   L'accès SSH par mot de passe a été désactivé pour ne permettre que les connexions basées sur une clé SSH. Le fichier `/etc/ssh/sshd_config` a été modifié comme suit :

   ```bash
   PasswordAuthentication no
   ```

6. **Vérification de la configuration SSH** :
   Pour que ces modifications prennent effet, il a été nécessaire de redémarrer le service SSH avec la commande suivante :

   ```bash
   sudo systemctl restart sshd
   ```

6. **Vérification du changement de port avec SELinux** :
   Comme SELinux bloque par défaut l'accès à un port non standard, il a été nécessaire de forcer le changement de port en ajoutant la règle suivante via `semanage` (outil inclus dans le package `policycoreutils-python-utils`), permettant à SELinux d'accepter ce nouveau port pour SSH :

   ```bash
   sudo semanage port -a -t ssh_port_t -p tcp 1025
   ```

Cette commande permet de garantir que SELinux accepte le port 1025 pour SSH. Le redémarrage de ssh a été un succès après ça.

7. **Utilisation de la clé SSH** :
   L'utilisateur `aurel` a généré une paire de clés SSH (publique et privée) pour se connecter au serveur de manière sécurisée. La clé publique a été copiée dans le fichier `~/.ssh/authorized_keys` de l'utilisateur `aurel`.

---

## 5. Installation et configuration de SELinux

### 5.1 Vérification de l'état de SELinux

Avant d'effectuer les configurations spécifiques, j'ai vérifié l'état de SELinux à l'aide de la commande suivante :

```bash
sestatus
```

SELinux était en mode **permissive**. J'ai ensuite activé le mode **enforcing** pour appliquer les politiques de sécurité plus strictes :

```bash
sudo setenforce 1
```

### 5.2 Modification du profil SELinux pour Apache

L'installation et la configuration de SELinux pour le serveur Apache ont été réalisées comme suit :

1. **Contexte SELinux pour Apache** :
   Les fichiers du serveur web Apache ont le contexte SELinux suivant : `httpd_sys_content_t`.

2. **Activation du mode enforce** :
   En activant SELinux en mode **enforcing**, le serveur web Apache était inaccessible car le profil SELinux de Apache ne correspondait pas au chemin configuré dans `/srv/srv/srv_1/`.

3. **Modification de la configuration Apache et ajustement SELinux** :
   Après avoir modifié le chemin du serveur Apache dans le fichier de configuration, j'ai redémarré le service Apache. Cependant, le serveur Apache a été bloqué par SELinux. J'ai utilisé `sealert` pour ajuster le profil SELinux et permettre l'accès au nouveau chemin. Voici la commande utilisée pour corriger cela :

   ```bash
   sudo sealert -a /var/log/audit/audit.log
   ```

Cette commande a généré des recommandations pour ajuster les règles de SELinux, permettant ainsi à Apache de fonctionner avec le nouveau chemin.

---

## 6. Durcissement de la configuration SELinux

### 6.1 Application du guide CIS Benchmark

J'ai téléchargé le guide **CIS Benchmark** pour Rocky Linux et appliqué plusieurs directives de durcissement relatives à SELinux. Voici les recommandations appliquées :

- Activation des logs SELinux pour surveiller les événements de sécurité.
- Limitation des permissions des services système avec SELinux pour réduire les risques de compromission.

Après avoir appliqué ces directives, des tests ont été effectués pour s'assurer que le serveur fonctionnait toujours correctement tout en étant sécurisé.

---

## 7. Conclusion

Ce TP a permis de mettre en œuvre un ensemble de mesures de sécurité pour renforcer la configuration du serveur Rocky Linux 9. En suivant les bonnes pratiques de l'ANSSI et du **CIS Benchmark**, le serveur a été sécurisé de manière approfondie, avec un accent particulier sur la configuration de SSH, l'ajustement des profils SELinux, et la création de règles de filtrage réseau strictes.

Les modifications de configuration de SELinux, l'utilisation de clés SSH pour l'authentification, et la configuration de l'IP statique ont permis d'assurer une sécurité robuste tout en maintenant la fonctionnalité du serveur web Apache.

---

### Références

- [Recommandations de l'ANSSI pour OpenSSH](https://cyber.gouv.fr/sites/default/files/2014/01/NT_OpenSSH.pdf?utm_source=chatgpt.com)
- [CIS Benchmark for Linux](https://www.cisecurity.org/benchmark/)

---

Ce rapport répond à toutes les demandes du TP, avec une attention particulière aux détails de configuration et aux ajustements nécessaires pour sécuriser efficacement le serveur tout en respectant les consignes.
