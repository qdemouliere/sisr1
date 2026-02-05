# Sécuriser l'accès au logiciel d'amorçage GRUB 2

## 1. Introduction

Le bootloader GRUB est un logiciel central lorsque l'on met en place un serveur sur GNU/Linux. En effet, il permet d'amorcer le démarrage du système en choisissant la version du noyau utilisée ainsi qu'un certain nombre d'options.

Or, par défaut n'importe qui peut modifier les paramètres d'amorçage de GRUB. Cela peut ainsi permettre à quelqu'un de malveillant de se connecter au système GNU/Linux en mode Single User qui correspond à mode de secours. Dans ce mode, on se connecte avec le superadministrateur sans mot de passe. On peut donc par la suite réaliser n'importe quelle opération sur l'OS.

Heureusement, il est possible d'imposer la mise en place d'une authentification par login et mot de passe lorsque que l'on souhaite modifier les paramètres de boot de GRUB.

## 2. Configuration

### 2.1 Définir un clavier azerty pour GRUB

Par défaut, GRUB ne reconnaît que le clavier en QWERTY, ce qui n'est pas très pratique lorsque l'on crée un utilisateur et un mot de passe avec un clavier AZERTY. Cependant, il est possible de modifier cette configuration par défaut.

On commence par créer le dossier */boot/grub/layouts*. Ensuite on génére la disposition du clavier dans un fichier reconnu par GRUB.

```bash
sudo mkdir /boot/grub/layouts
sudo grub-kbdcomp -o /boot/grub/layouts/fr.gkb fr
```

Il est nécessaire de définir ce changement de type de clavier dans les fichiers de configuration du bootloader.

```bash
sudoedit /etc/default/grub

GRUB_TERMINAL_INPUT=at_keyboard 
```

```bash
sudoedit /etc/grub.d/40_custom

# Clavier fr
insmod keylayouts
keymap fr
```

Il est possible de mettre à jour ces changements.

```bash
sudo update-grub
```

### 2.2 Définition d'un utilisateur et d'un mot de passe

Dans un second temps, il faut créer un utilisateur et un mot de passe afin de s'authentifier si l'on souhaite modifier les paramètres d'amorçage.

La première étape consiste à obtenir le hash d'un mot de passe saisi à l'aide de la commande *grub-mkpasswd-pbkdf2* 

```bash
grub-mkpasswd-pbkdf2
Enter password:
Reenter password:
```

Il faut ensuite fournir ce hash et un nom d'utilisateur dans le fichier */etc/grub.d/40_custom*.

```bash
sudoedit /etc/grub.d/40_custom

# Défintion d'un utilisateur pour grub
set superusers=adminsio
# Copier le mot de passe haché généré précédemment
password_pbkdf2 adminsio grub.pbkdf2.sha512.10000.C0F70D240A8BC5F[…]
```

Il reste enfin à prendre en compte ces nouveaux paramètres et à redémarrer.

```bash
sudo update-grub
sudo shutdown -r now
```

## 3.Conclusion

Dorénavant, si vous tentez de modifier la séquence de boot à l'aide de la touche e (edit), une authentification vous sera demandée et il ne sera plus possible de démarrer en mode single user sans y être
