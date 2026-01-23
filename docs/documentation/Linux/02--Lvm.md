# LVM le gestionnaire de volumes logiques sous GNU/Linux

## 1. Introduction

La gestion des disques et de l'espace de stockage peut vite devenir un casse-tête sur un serveur notamment lorsque l'on se rend compte après coup que le partitionnement choisi sur le disque dur n'était pas pertinent ou que l'une des paritions est saturée.

Heureusement sur GNU/Linux, une solution géniale existe ! Elle se nomme **Logical Volume Manager** (LVM). Elle pourra vous sauver la mise dans bien des situations stressantes et compliquées.

## 2. Historique

 À la fin des années 1990, IBM a libéré le code source de son gestionnaire de volumes logiques (LVM) et de son système de fichiers journalisé (JFS) afin de les porter sur Linux. Cette initiative a permis d’intégrer LVM dans le noyau Linux, offrant ainsi une solution flexible pour la gestion des espaces de stockage, notamment la possibilité de redimensionner dynamiquement les partitions et de créer des volumes s’étendant sur plusieurs disques physiques. 

Puis LVM 2 a été proposée et ajoute des fonctionnalités supplémentaires extrêmement puissantes telles que les snapshots et le thin provisioning (création de volumes n'utilisant que l'espace en cours d'utilisation, ex : un volume logique de 30 Go qui ne disposera que de 8Go d'espace utilisé aura une taille de 8 Go et pourra augmenter dynamiquement).

LVM 2 est un outil présent sur la majorité des distributions GNU/Linux modernes dont Debian.

## 3. Comment fonctionne LVM 2 ?

Voici les éléments clés de l’architecture de LVM2 :

- **Physical Volumes (PV)** : Ce sont les disques physiques ou partitions qui serviront de base pour LVM2. Ils peuvent être de différents types (HDD, SSD, etc.).

- **Volume Groups (VG)** : Un Volume Group regroupe plusieurs Physical Volumes en un seul espace logique. C’est une sorte de “pool” dans lequel on peut puiser pour créer des volumes logiques.

- **Logical Volumes (LV)** : Les Logical Volumes sont des morceaux du Volume Group. Ce sont eux que vous formatez et montez pour y stocker vos données. Ils fonctionnent comme des partitions, mais avec beaucoup plus de flexibilité.

- **Thin Provisioning** : C’est une fonctionnalité avancée qui permet de créer des volumes logiques sans allouer immédiatement tout l’espace. Pratique pour optimiser l’utilisation des ressources.

- **Snapshots** : Ce sont des copies instantanées d’un volume logique. Parfait pour sauvegarder ou tester des modifications.

Voici un schéma de principe expliquant le fonctionnement de LVM 2 :

![](../../media/linux/lvm.png)
## 4. Les commandes liées à l'utilisation de LVM 2

**Installer LVM 2 sur Debian si nécessaire**

```bash
sudo apt update
sudo apt install lvm2
```

**Analyse des disques existants sur le système**

```bash 
 sudo lsblk
```

On peut également utiliser l'outil historique *fdisk*.

```bash
sudo fdisk -l 
```


**Convertir des disques ou des partitions en Physical Volumes (PV)**

```bash
sudo pvcreate /dev/sda /dev/sdb /dev/sdc /dev/sdd

  Physical volume "/dev/sda" successfully created.
  Physical volume "/dev/sdb" successfully created.
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
```

Si l'on souhaite initialiser un seul disque ou une seule partition :

```bash
sudo pvcreate sde
sudo pvcreate sde1
```

**Obtenir les informations concernant les PVE**

```bash
sudo pvdisplay
```

**Création d'un Volume Group (VG)**

```bash
sudo vgcreate Data /dev/sda /dev/sdb /dev/sdc /dev/sdd
  Volume group "Data" successfully created
```

**Obtenir les informations concernant les VG** 

```bash
sudo vgdisplay
```

**Création de volumes logiques (LV)** 

```bash
sudo lvcreate -L 250G -n MySQLData Data
```

**Obtenir les informations concernant les LV** 
```bash

sudo lvdisplay
```

**Formatage des volumes logiques (LV)**

```bash
sudo mkfs.ext4 /dev/Data/MySQLData
```

**Montage des volumes logiques sur le système**

```bash
sudo mkdir -p /bdd
sudo mount /dev/Data/MySQLData /bdd
```

Si vous souhaitez rendre les points de montage persistants (automatiquement montés au démarrage de l'OS), il est nécessaire d'ajouter une ligne spécifique dans le fichier /etc/fstab. 

Pour vérifier si le montage est effectif, vous pouvez utiliser les commandes df ou mount.

**Augmentation de la taille d'un volume logique** 

```bash
sudo lvextend -L +20G /dev/Data/MySQLData
```

Après avoir agrandi le volume logique, il est nécessaire de redimensionner le système de fichier utilisé

```bash
sudo resize2fs /dev/Data/MySQLData
```

**Réduction de la taille d'un volume logique**

!!! Warning  "Attention"
    La réduction d'un volume logique n'est pas anodin et peut entrainer des problèmes dont la perte ou la corruption de données. Réalisez cette opération avec prudence.

```bash
sudo resize2fs /dev/Data/MySQLData 30G
sudo lvreduce -L 30G /dev/Data/MySQLData
```

**Suppression d'un volume logique**

```bash
sudo umount /bdd 
sudo lvremove /dev/Data/MySQLData 
```

**Suppression d'un groupe de volume** 

```bash 
sudo lvremove /dev/Data
sudo vgremove Data
```

**Surveiller l'état des volumes** 

```bash 
sudo lvscan
```

## 5. Les snapshots

LVM permet de réaliser des clichés instantanés de volumes logiques (snapshots). Les snapshots sont des volumes logiques "spéciaux". Ce LV "snapshot" ne contient pas de système de fichiers et ne peut donc pas être monté. Il va stocker uniquement les modifications de données (ajout/suppressions). On définira une taille à ce volume logique de type snapshot, mais cette valeur n'agrandira pas le volume logique initial évidemment

Dans un premier temps, il faut s'assurer qu'il reste de la place dans le groupe de volume afin de pouvoir créer un nouveau volume logique dédié aux snpashots.

```bash 
sudo vgs
```

Il faut identifier ensuite le volume logique pour lequel on veut créer un instantané.

```bash 
sudo lvs 
```

**Création du snapshot**

```bash 
sudo lvcreate -L 10G -s -n snapbdd /dev/Data/MySQLData
```

On va créer ici un nouveau volume logique de 10 Giga dédié aux snapshots de MySQLData. Les 10 Giga ne seront pas directement mobilisés. Au fur et à mesure des modifications dans MySQLData, *snpbdd* grossira et utilisera de plus en plus d'espace libre.

La taille va augmenter à chaque modification sur le système de fichier concerné : création, modification, suppression de fichiers. Il peut donc théoriquement avoir besoin de plus de place sur le LV "snapshot" que de place initiale sur le LV source. Il s'agit du même mécanisme que les snapshots de machine virtuelle.

**Restaurer un snapshot démontable**

Si l'on ne souhaite pas conserver les modifications réalisées après le snapshot, on va restaurer le volume logique dans son état antérieur.

```bash 
sudo umount /bdd 
sudo lvconvert --mergesnapshot /dev/Data/MySQLData
sudo mount /data 
```

**Restaurer un snapshot non démontable** 

Dans le cas d'un snapshot de la racine par exemple, ou d'un volume utilisé par des processus qu'on ne peut stopper, on va lancer directement la commande lvconvert.

```bash 
sudo lvconvert --mergesnapshot /dev/Data/snapbdd
sudo shutdown -r shutdown
```

Pour prendre en compte la restauration, nous n'aurons pas d'autre choix que de redémarrer la machine.

**Fusionner le snapshot** 

Si après des modifications, on souhaite garder les données en l'état, on va supprimer le snapshot réalisé précédemment. Le LV d'origine et le LV snapshot seront fusionnés.

```bash 
sudo lvremove /dev/Data/snapbdd
```

## Sources 

- [La documentation sur LVM de Stéphane Robert](https://blog.stephane-robert.info/docs/admin-serveurs/linux/lvm/) 
- [L'article sur les snapshots LVM d'Adrien Linux Tricks](https://www.linuxtricks.fr/wiki/lvm-avance-les-snapshots) 
- [La documentation sur LVM d'IT Connect](https://www.it-connect.fr/gestion-des-lvm-sous-linux/) 

