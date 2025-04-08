# **TP 2: Sécurisation d'un serveur web avec SELinux et durcissement**

## **1. Installation du système d’exploitation**

### **1.1 Objectif**
L'objectif de cette première étape est d'installer un serveur Rocky Linux 9 minimal, sans interface graphique, et de s'assurer que les derniers correctifs de sécurité sont appliqués.

### **1.2 Installation du système**
L'installation de **Rocky Linux 9** a été réalisée à partir de l'image ISO officielle en choisissant l'option **Minimal Install** afin de ne pas inclure d'interface graphique.

Une fois l'installation du système de base terminée, il convient de vérifier que le système est à jour en exécutant la commande suivante :
```bash
sudo dnf update -y
```
Cette commande permet de télécharger et d'installer les derniers correctifs de sécurité. Le temps nécessaire pour cette mise à jour peut varier entre 10 et 30 minutes, en fonction de la vitesse de la connexion et du nombre de mises à jour disponibles.


---

## **2. Sécurisation de l’administration du serveur**

### **2.1 Objectif**
Cette étape consiste à renforcer la sécurité de l'administration du serveur via SSH, en interdisant l'accès root, en désactivant l'authentification par mot de passe, et en autorisant uniquement l'administrateur avec une clé SSH.

### **2.2 Configuration de SSH**
Pour sécuriser l'administration du serveur, il est nécessaire de configurer le serveur SSH afin de :

- Désactiver l'accès à l'utilisateur root,
- Interdire l'authentification par mot de passe et ne permettre que l'authentification par clé SSH,
- Restreindre l'accès SSH uniquement à un utilisateur administrateur.
- Changer le port de SSH

Les étapes suivantes ont été appliquées :
1. Modification du fichier de configuration SSH `/etc/ssh/sshd_config` pour inclure les lignes suivantes :
    ```bash
    Port 1025
    PermitRootLogin no
    PasswordAuthentication no
    PubkeyAuthentication yes
    AllowUsers aurel
    ```
2. Redémarrage du service SSH pour que les modifications prennent effet :
    ```bash
    sudo systemctl restart sshd
    ```

### **2.3 Filtrage réseau des flux entrants et sortants**
Le filtrage strict du trafic réseau a été configuré en utilisant **firewalld** :
1. Ajout des services SSH et HTTP au pare-feu pour permettre uniquement le trafic nécessaire :
    ```bash
    sudo firewall-cmd --zone=public --add-service=ssh --permanent
    sudo firewall-cmd --zone=public --add-service=http --permanent
    sudo firewall-cmd --reload
    ```

### **2.4 Temps estimé**
- **Durée estimée pour cette étape : 30 minutes à 1 heure** (comprend la configuration de SSH et du pare-feu).

---

## **3. Installation et configuration du serveur web Apache**

### **3.1 Objectif**
L'objectif est d'installer le serveur web **Apache** et de tester son fonctionnement de base avant d'appliquer SELinux.

### **3.2 Installation d'Apache**
Apache a été installé en utilisant la commande suivante :
```bash
sudo dnf install httpd -y
```
Ensuite, le service Apache a été activé et démarré :
```bash
sudo systemctl enable --now httpd
```

### **3.3 Test d'accès au serveur Apache**
Le serveur Apache a été testé en accédant à l'adresse `http://<ip_du_serveur>` depuis un navigateur pour vérifier que le serveur web fonctionne correctement avec la configuration par défaut.

### **3.4 Installation de SELinux**
Si SELinux n'était pas déjà installé, il a été installé en utilisant la commande suivante :
```bash
sudo dnf install selinux-policy-targeted
```

### **3.5 Temps estimé**
- **Durée estimée pour cette étape : 30 minutes à 1 heure** (comprend l'installation d'Apache et le test d'accès, ainsi que l'installation de SELinux si nécessaire).

---

## **4. Modification du profil SELinux pour Apache**

### **4.1 Objectif**
L'objectif est d'appliquer des règles SELinux pour Apache et tester son fonctionnement avec des profils SELinux modifiés.

### **4.2 Contextes des fichiers du serveur web Apache**
Les contextes de fichiers sont associés à chaque fichier ou répertoire sur le serveur, et chaque service ou utilisateur peut avoir un contexte différent. Pour Apache, les fichiers du serveur web doivent avoir des contextes spécifiques. Cela peut être vérifié avec la commande :
```bash
ls -Z /var/www/html
```

### **4.3 Activation du mode « enforce » et tests**
Le mode **"enforce"** de SELinux a été activé pour Apache :
```bash
sudo setenforce 1
```
À ce stade, il a été constaté si le serveur Apache est toujours accessible. Si le serveur ne fonctionne pas, il peut être nécessaire d'ajuster les politiques SELinux.

### **4.4 Modification de la configuration d'Apache**
Le path du serveur web a été modifié pour se situer dans `/srv/srv/srv_1/`. Ensuite, le service Apache a été redémarré :
```bash
sudo systemctl restart httpd
```

### **4.5 Ajustement du profil SELinux**
Après avoir modifié la configuration du serveur Apache, un message d'erreur SELinux peut apparaître si les profils ne correspondent pas à la nouvelle configuration. L'outil `sealert` peut être utilisé pour examiner les alertes SELinux et ajuster les politiques :
```bash
sudo sealert -a /var/log/audit/audit.log
```
Cet outil fournit des suggestions sur les ajustements à apporter aux profils SELinux pour permettre à Apache de fonctionner avec les nouvelles configurations.

### **4.6 Tests après modification du profil**
Une fois les ajustements effectués, SELinux a été activé à nouveau en mode **enforce**, et l'accès au serveur Apache a été vérifié depuis le navigateur.

### **4.7 Temps estimé**
- **Durée estimée pour cette étape : 1 à 2 heures** (comprend la modification du profil SELinux, les tests et l'ajustement des configurations).

---

## **5. Durcissement de la configuration de SELinux**

### **5.1 Objectif**
Renforcer la configuration de SELinux en suivant les recommandations du CIS Benchmark pour SELinux sur Rocky Linux.

### **5.2 Application des recommandations CIS**
Le **CIS Benchmark** pour SELinux a été consulté pour appliquer les recommandations de durcissement appropriées. Cela peut inclure :
- L'activation de la politique de contrôle d'accès la plus stricte,
- La désactivation des services non utilisés,
- La révision des politiques de gestion des contextes et des labels SELinux.

### **5.3 Tests de fonctionnalité**
Après avoir appliqué les recommandations de durcissement, il est essentiel de vérifier que les services sont toujours accessibles et fonctionnels. Cela peut impliquer des tests d'accès à Apache et à d'autres services pour s'assurer qu'ils ne sont pas affectés par les nouvelles configurations de SELinux.

### **5.4 Temps estimé**
- **Durée estimée pour cette étape : 1 heure** (comprend l'application des directives du CIS Benchmark et les tests de fonctionnalité).

---

## **Temps total estimé**
- **Temps total estimé pour le TP : 4 à 5 heures**, en fonction de la complexité des ajustements de SELinux et des tests nécessaires.

---

### **Conclusion**
Le TP a permis de configurer et de durcir un serveur Rocky Linux 9 en mettant en œuvre des mesures de sécurité comme la configuration de SSH, le filtrage réseau, l'installation et la configuration d'Apache, et l'utilisation de SELinux pour renforcer la sécurité. La mise en place de profils SELinux spécifiques et le durcissement via le CIS Benchmark ont permis de renforcer la sécurité du serveur tout en maintenant son fonctionnement normal.
