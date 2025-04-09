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

7. **Vérification du changement de port avec SELinux** :
   Comme SELinux bloque par défaut l'accès à un port non standard, il a été nécessaire de forcer le changement de port en ajoutant la règle suivante via `semanage` (outil inclus dans le package `policycoreutils-python-utils`), permettant à SELinux d'accepter ce nouveau port pour SSH :

   ```bash
   sudo semanage port -a -t ssh_port_t -p tcp 1025
   ```

Cette commande permet de garantir que SELinux accepte le port 1025 pour SSH. Le redémarrage de ssh a été un succès après ça.

8. **Utilisation de la clé SSH** :
   L'utilisateur `aurel` a généré une paire de clés SSH (publique et privée) pour se connecter au serveur de manière sécurisée. La clé publique a été copiée dans le fichier `~/.ssh/authorized_keys` de l'utilisateur `aurel`.



### 5. Installation d’un serveur Web

1. **Installation du serveur web Apache**

Le serveur web Apache est installé à l’aide de la commande suivante :

```bash
sudo dnf install httpd -y
```

Cette commande installe le paquet `httpd`, qui est le nom correct du serveur Apache sous Rocky Linux. Le paquet `apache` n’existe pas dans les dépôts standards, ce qui explique l'erreur initiale obtenue avec la commande `sudo dnf install apache -y`.

Une fois le service installé, il est démarré et activé au démarrage :

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

2. **Installation de SELinux**

Sur Rocky Linux, SELinux est installé et activé par défaut. La commande suivante permet de vérifier son état :

```bash
sestatus
```
![image](https://github.com/user-attachments/assets/01be613a-14fd-424e-ab90-d99d5b1d60f5)

3. **Les différents modes de SELinux**

SELinux dispose de trois modes de fonctionnement :

- `enforcing` : les politiques de sécurité sont appliquées et les violations sont bloquées.
- `permissive` : les violations sont simplement enregistrées dans les logs, mais non bloquées.
- `disabled` : SELinux est désactivé.

4. **Comportement en mode enforcing avec un profil inadéquat**

Si un binaire n’est pas conforme à la politique SELinux en mode `enforcing`, son exécution sera bloquée. Cela empêche tout comportement potentiellement dangereux, mais peut aussi bloquer certains services légitimes mal configurés.

---

### 3.4 Modification d’un profil SELinux

1. **Contexte des fichiers Apache**

Pour consulter le contexte des fichiers du serveur web, la commande suivante est utilisée :

```bash
ls -Z /var/www/html/
```

![image](https://github.com/user-attachments/assets/5d7113f7-165e-4ae1-9660-6c519b953ccf)

2. **Contexte du service Apache**

Le contexte du service Apache peut être vérifié avec :

```bash
ps -eZ | grep httpd
```

![image](https://github.com/user-attachments/assets/b3dd35ab-7902-4c63-bf2f-708db9a26bce)

3. **Activation du mode enforcing**

Le mode `enforcing` est activé avec la commande suivante :

```bash
sudo setenforce 1
```

Un test d’accès au serveur est ensuite effectué depuis un navigateur. Si le serveur est mal configuré, l’accès échoue.

![image](https://github.com/user-attachments/assets/277ff5b3-351d-4953-834b-744ed461abd8)

Alors on peut conclure qu'il est bien configuré

4. **Déplacement du serveur web vers /srv/srv/srv_1/**

Le mode enforcing est temporairement désactivé :

```bash
sudo setenforce 0
```

La configuration d’Apache est modifiée dans `/etc/httpd/conf/httpd.conf` pour changer le `DocumentRoot` :

```apacheconf
DocumentRoot "/srv/srv/srv_1"
<Directory "/srv/srv/srv_1">
    AllowOverride None
    Require all granted
</Directory>
```

Le contenu web est déplacé :

```bash
sudo mkdir -p /srv/srv/srv_1
sudo cp /var/www/html/index.html /srv/srv/srv_1/
```

Puis le service Apache est redémarré :

```bash
sudo systemctl restart httpd
```

5. **Activation du mode enforcing et test d’accès**

Le mode enforcing est de nouveau activé :

```bash
sudo setenforce 1
```

![image](https://github.com/user-attachments/assets/daf7f64c-74a6-4efe-8069-b366805bafee)


L’accès échoue car le nouveau chemin n’a pas les bons contextes SELinux.

6. **Ajustement du profil SELinux avec `sealert`**

La commande suivante permet d’identifier les erreurs SELinux :

```bash
sudo sealert -a /var/log/audit/audit.log
```

`sealert` fournit des recommandations. Par exemple, pour réétiqueter les fichiers avec le bon contexte :

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/srv/srv_1(/.*)?"
sudo restorecon -Rv /srv/srv/srv_1
```

`sealert` est utile pour détecter automatiquement les violations de politiques SELinux, proposer des correctifs, et aider à appliquer les bons contextes de sécurité.

7. **Vérification finale**

Une fois le bon contexte appliqué et SELinux en mode `enforcing`, le serveur Apache est de nouveau accessible via le navigateur.

---

### 3.5 Durcissement de la configuration de SELinux

Pour durcir SELinux, le guide CIS Benchmark pour Rocky Linux est utilisé comme référence. Quelques recommandations appliquées :

- S'assurer que SELinux est activé et en mode `enforcing` au démarrage :

```bash
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
```

- Supprimer les modules inutiles avec :

```bash
sudo semodule -l
sudo semodule -r nom_du_module
```

- Activer l’audit des événements SELinux dans `auditd`.

Après ces modifications, le système est redémarré, puis il est vérifié que les services web et SSH fonctionnent correctement avec SELinux activé en mode `enforcing`, et que les contextes appliqués sont corrects.

Tous les profils fonctionnent comme attendu après ce durcissement.

