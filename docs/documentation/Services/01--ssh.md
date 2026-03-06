# Administration et configuration d'un service OpenSSH

## I. Introduction à OpenSSH

OpenSSH est une suite d’outils open source permettant de sécuriser les connexions à distance entre machines via le protocole SSH (Secure Shell). Il remplace les protocoles non chiffrés comme Telnet ou FTP, en offrant un chiffrement fort pour toutes les communications, empêchant ainsi l’écoute, le détournement de connexion et d’autres attaques. OpenSSH est intégré par défaut dans la plupart des distributions Linux, dont Debian 13, et est largement utilisé pour l’administration système, le transfert de fichiers sécurisé (SCP, SFTP) et la création de tunnels chiffrés.

**Fonctionnalités clés :**
- Chiffrement de bout en bout
- Authentification par mot de passe ou par clé publique/privée
- Support des algorithmes modernes (Ed25519, ChaCha20-Poly1305)
- Gestion centralisée des clés et des connexions


## II. Fonctionnement d’une Connexion SSH

### 1. Principe de Base

Une connexion SSH s’établit entre un **client** (votre machine) et un **serveur** (la machine distante). Le protocole SSH utilise un mécanisme de chiffrement asymétrique pour authentifier le serveur et, optionnellement, le client. Voici les étapes principales :

1. **Échange de clés** : Le client et le serveur négocient une clé de session temporaire (échange de clés de type Diffie-Hellman).
2. **Authentification du serveur** : Le client vérifie l’identité du serveur via sa clé publique.
3. **Authentification du client** : Le client s’authentifie (par mot de passe ou clé SSH).
4. **Ouverture d’un canal sécurisé** : Toutes les données transitent chiffrées.

### 2. Approche "Trust on First Use" (TOFU)

OpenSSH utilise le modèle **Trust on First Use** : lors de la première connexion à un serveur, le client enregistre automatiquement la clé publique du serveur dans le fichier `~/.ssh/known_hosts`. Cette clé servira à vérifier l’identité du serveur lors des connexions suivantes.

**Exemple :**
```bash
The authenticity of host 'serveur.example.com (192.168.1.10)' can't be established.
ECDSA key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
``` 


**Ajout de la clé dans `known_hosts`**

Si vous répondez **yes**, la clé du serveur est ajoutée au fichier `known_hosts`.

### 3. Le fichier `~/.ssh/known_hosts`

Ce fichier stocke les **clés publiques des serveurs** auxquels vous vous êtes connecté.

Chaque entrée contient :

- L’adresse IP ou le nom d’hôte du serveur
- Le type de clé (ex : `ecdsa-sha2-nistp256`)
- La clé publique elle-même

!!! warning "Attention"
    Depuis plusieurs années, le contenu du fichier *known_hosts* n'est plus lisible via un éditeur de texte ou la commande *cat* pour des raisons de sécurité. Il faudra passer par la commande ssh-keygen pour gérer les entrées présentes à l'intérieur.


### Exemple de contenu

```text
serveur.example.com,192.168.1.10 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBB...
```

!!! warning "Attention"
    Si la clé du serveur change (ex : réinstallation), SSH refusera la connexion afin d'éviter les attaques **Man-in-the-Middle**.  
    Il faudra alors supprimer l’ancienne entrée dans `known_hosts`.

Voici le nouveau format que vous pourriez retrouver dans ce fichier :

```text 
|1|pz3StX4JbD/XviB6ETOYlcaFqLA=|L0IubZCEOrV/VWE9qR4GzoUThSU= ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNBGU/vaa6OdX8kIxle0aom7wu8rbIMaTZo+Q+lrVoZf1xsw0odlojFsoVjsOarRZ4oP9Ywf9wjya6dFNYBndHo=
```

### 4. Commandes SSH de Base

| Commande | Description |
|---------|-------------|
| `ssh utilisateur@serveur` | Se connecter à un serveur |
| `ssh -p 2222 utilisateur@serveur` | Se connecter sur un port personnalisé |
| `ssh -i ~/.ssh/ma_cle_privee utilisateur@serveur` | Utiliser une clé SSH spécifique |
| `scp fichier utilisateur@serveur:/chemin/destination` | Copier un fichier vers le serveur |
| `sftp utilisateur@serveur` | Ouvrir une session SFTP interactive |


## II. Configuration du Serveur SSH sous Debian 13

### 1. Installation et Activation du Service

Sur **Debian 13**, le serveur OpenSSH s’installe via :

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl restart ssh 
sudo systemctl enable --now ssh
```

Vérifiez le statut du service :

```bash
sudo systemctl status ssh
```


### 2. Fichier de Configuration `/etc/ssh/sshd_config`

Ce fichier permet de configurer le comportement du serveur SSH.

| Directive | Valeur recommandée | Description |
|----------|-------------------|-------------|
| `Port` | `22` (ou personnalisé) | Port d’écoute du serveur |
| `PermitRootLogin` | `no` | Interdit la connexion en tant que root |
| `PasswordAuthentication` | `no` | Désactive l’authentification par mot de passe |
| `PubkeyAuthentication` | `yes` | Active l’authentification par clé |
| `X11Forwarding` | `no` | Désactive le transfert X11 |
| `AllowUsers` | `utilisateur1 utilisateur2` | Limite les utilisateurs autorisés |

**Exemple de configuration sécurisée**

```ini
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers alice bob
```

**Bonnes pratiques**

Faire une sauvegarde avant modification :

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

Redémarrer le service après modification :

```bash
sudo systemctl restart ssh
```


### 3. Les clés présentes sur le serveur : `/etc/ssh/ssh_host_*`

Le serveur dispose de plusieurs paires de clés, en général 3. En fonction du client qui sollicite la connexion et des algorithmes qu'il supporte, le serveur s'adaptera et utilisera la paire de clés la plus adaptée au client.

Contrairement à ce que l'on pourrait croire de prime abord, ces paires de clés ne servent pas au chiffrement (qui sera assuré par un clé secrète symétrique générée à partir de l'algorithme d'échange de clés Diffie-Hellman) mais à prouver l'identité du serveur. Ce dernier fournira l'empreinte de sa clé publique au client lors de la première connexion.

| Fichier | Description |
|-------|-------------|
| `ssh_host_ed25519_key` | Clé privée Ed25519 (recommandée) |
| `ssh_host_ed25519_key.pub` | Clé publique Ed25519 |
| `ssh_host_rsa_key` | Clé privée RSA (obsolète, à éviter) |

**Pourquoi Ed25519 ?**

- Plus sécurisé et performant que RSA ou DSA
- Recommandé par les bonnes pratiques actuelles

!!! danger "Sécurité"
    **Ne jamais partager la clé privée du serveur.**


## III. Authentification par Clés SSH (Ed25519)

### 1. Limites de l’Authentification par Mot de Passe

- Vulnérabilité aux attaques par **force brute**
- Exposition aux **fuites de mots de passe**
- Absence de **traçabilité fiable**


### 2. Génération d’une Paire de Clés Ed25519

Sur votre machine locale :

```bash
ssh-keygen -t ed25519 -C "votre_email@example.com"
```

Cela génère :

- Clé privée : `~/.ssh/id_ed25519`
- Clé publique : `~/.ssh/id_ed25519.pub`

!!! tip "Astuce"
    Protégez votre clé privée avec une **phrase de passe (passphrase)**.


### 3. Copie de la Clé Publique sur le Serveur

**Méthode recommandée**

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub utilisateur@serveur
```

!!! warning "Attention"
   Lors du transfert de votre clé publique sur le serveur via cette commande, vous devez vous assurer que l'authentification par login
   et mot de passe est toujours active pour pouvoir réaliser l'opération.

**Méthode manuelle**

```bash
cat ~/.ssh/id_ed25519.pub | ssh utilisateur@serveur "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### 4. Désactivation de l’authentification par mot de passe

Modifier `/etc/ssh/sshd_config` :

```ini
PasswordAuthentication no
PubkeyAuthentication yes
```

Puis redémarrer le service :

```bash
sudo systemctl restart ssh
```

### 5. Vérification et dépannage

Tester la connexion :

```bash
ssh -i ~/.ssh/id_ed25519 utilisateur@serveur
```

Vérifier les permissions :

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Consulter les logs :

Avec journactl

```bash
sudo journalctl -u ssh --no-pager -n 50
```


Avec rsyslog

```bash
sudo cat /var/log/auth.log
```


## IV Conclusion

**OpenSSH** est un outil indispensable pour l’administration sécurisée de serveurs distants.  
En appliquant ces bonnes pratiques, vous réduisez fortement les risques de compromission.



