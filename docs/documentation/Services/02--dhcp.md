# Installer et configurer un serveur DHCP ave KEA sur Debian

## Sources

- Documentation technique produite par D. Balny - BTS SIO
- https://kea.readthedocs.io/en/kea-2.6.1/arm/dhcp4-srv.html
- https://www.it-connect.fr/linux-installer-et-configurer-un-serveur-dhcp-kea-sur-debian/
- https://www.it-connect.fr/serveur-dhcp-sous-linux/

---

# I. Présentation

Cette documentation permet d'apprendre à **installer et configurer un serveur DHCP sous Debian** à l'aide de la solution **ISC KEA DHCP Server**.

ISC KEA est le **successeur du serveur ISC DHCP Server** et propose une architecture plus moderne et évolutive.

## Composants principaux

| Paquet | Description |
|------|------|
| isc-kea-dhcp4 | Serveur DHCP IPv4 |
| isc-kea-dhcp6 | Serveur DHCP IPv6 |
| isc-kea-dhcp-ddns | Mise à jour DNS dynamique |
| isc-kea-admin | Outils d'administration |

!!! info
    KEA utilise un **fichier de configuration au format JSON**.

!!! tip
    KEA peut également utiliser une base de données :

    - MySQL  
    - PostgreSQL  
    - Cassandra  

Dans ce tutoriel nous utiliserons **le mode par défaut avec stockage en mémoire et fichier CSV**.

---

# Objectif

Configurer un serveur DHCP capable de distribuer :

- une plage d’adresses IP
- un serveur DNS
- une passerelle par défaut
- une réservation DHCP

Plage d'adresses :

```
192.168.14.100 → 192.168.14.120
```

---

# Prérequis

- Une machine **Debian avec une adresse IP fixe**

```
192.168.14.99/24
```

- Aucun autre serveur DHCP actif sur le réseau
- Une machine cliente pour les tests
- Une connexion Internet

!!! warning
    Plusieurs serveurs DHCP sur un même réseau peuvent provoquer des conflits.

---

# II. Installation du serveur DHCP KEA

Mettre à jour les paquets :

```bash
sudo apt-get update
```

Installer le serveur DHCP :

```bash
sudo apt-get install kea-dhcp4-server
```

Vérifier le service :

```bash
sudo systemctl status kea-dhcp4-server
```

Exemple :

```text
● kea-dhcp4-server.service - Kea IPv4 DHCP daemon
Loaded: loaded (/lib/systemd/system/kea-dhcp4-server.service)
Active: active (running)
Main PID: 3427 (kea-dhcp4)
```

!!! note
    Le fichier de configuration utilisé est :

```
/etc/kea/kea-dhcp4.conf
```

---

# III. Configuration du serveur DHCP KEA

## Identifier l’interface réseau

Afficher la configuration réseau :

```bash
ip a
```

Exemple d’interface :

```
ens33
```

---

## Sauvegarde de la configuration

Avant modification :

```bash
sudo mv /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.bkp
```

Créer un nouveau fichier :

```bash
sudo nano /etc/kea/kea-dhcp4.conf
```

---

# Configuration de base

```json
{
 "Dhcp4": {

    "interfaces-config": {
        "interfaces": [ "ens33" ]
    },

    "valid-lifetime": 691200,
    "renew-timer": 345600,
    "rebind-timer": 604800,

    "authoritative": true,

    "lease-database": {
        "type": "memfile",
        "persist": true,
        "name": "/var/lib/kea/kea-leases4.csv",
        "lfc-interval": 3600
    }

 }
}
```

---

# Explications

## Interface réseau

```json
"interfaces-config": {
 "interfaces": ["ens33"]
}
```

Interface utilisée par le serveur DHCP.

---

## Durée des baux DHCP

```json
"valid-lifetime": 691200
```

Durée du bail : **8 jours**

```json
"renew-timer": 345600,
"rebind-timer": 604800
```

| Paramètre | Description |
|------|------|
| renew-timer | renouvellement à 50% |
| rebind-timer | renouvellement à 87,5% |

---

## Serveur autoritaire

```json
"authoritative": true
```

!!! note
    Le serveur renverra un **DHCP NAK** si une adresse IP demandée ne correspond pas au réseau géré.

---

## Base de données des baux

```json
"lease-database": {
 "type": "memfile",
 "persist": true,
 "name": "/var/lib/kea/kea-leases4.csv",
 "lfc-interval": 3600
}
```

| Paramètre | Description |
|------|------|
| memfile | stockage mémoire |
| persist | sauvegarde active |
| csv | fichier de stockage |
| lfc-interval | nettoyage automatique |

---

# B. Création d’une étendue DHCP

Ajouter dans la configuration :

```json
"subnet4": [
 {
  "subnet": "192.168.14.0/24",
  "pools": [
   { "pool": "192.168.14.100 - 192.168.14.120" }
  ],
  "option-data": [
   {
    "name": "domain-name-servers",
    "data": "192.168.14.201"
   },
   {
    "name": "domain-search",
    "data": "it-connect.local"
   },
   {
    "name": "routers",
    "data": "192.168.14.2"
   }
  ]
 }
]
```

Options DHCP :

| Option | Description |
|------|------|
| domain-name-servers | serveur DNS |
| domain-search | domaine |
| routers | passerelle |

---

# Configuration complète

```json
{
 "Dhcp4": {
  "interfaces-config": {
   "interfaces": ["ens33"]
  },

  "valid-lifetime": 691200,
  "renew-timer": 345600,
  "rebind-timer": 604800,

  "authoritative": true,

  "lease-database": {
   "type": "memfile",
   "persist": true,
   "name": "/var/lib/kea/kea-leases4.csv",
   "lfc-interval": 3600
  },

  "subnet4": [
   {
    "subnet": "192.168.14.0/24",
    "pools": [
     {
      "pool": "192.168.14.100 - 192.168.14.120"
     }
    ],
    "option-data": [
     {
      "name": "domain-name-servers",
      "data": "192.168.14.201"
     },
     {
      "name": "domain-search",
      "data": "it-connect.local"
     },
     {
      "name": "routers",
      "data": "192.168.14.2"
     }
    ]
   }
  ]
 }
}
```

Redémarrer le service :

```bash
sudo systemctl restart kea-dhcp4-server.service
```

---

# Vérification des erreurs

Afficher les logs :

```bash
sudo journalctl -xe | grep kea
```

Exemple d’erreur :

```text
DHCP4_INIT_FAIL failed to initialize Kea server
syntax error
```

!!! tip
    Vérifier la syntaxe JSON :

    https://jsonlint.com

---

# C. Réservation DHCP

Une réservation DHCP permet d’associer **une adresse IP à une adresse MAC**.

| Paramètre | Valeur |
|------|------|
| IP | 192.168.14.100 |
| MAC | 00:0c:29:0a:6f:c3 |
| Machine | Ubuntu2404 |

Configuration :

```json
"reservations": [
 {
  "hw-address": "00:0c:29:0a:6f:c3",
  "ip-address": "192.168.14.100",
  "hostname": "Ubuntu2404"
 }
]
```

Redémarrer le service :

```bash
sudo systemctl restart kea-dhcp4-server.service
```

!!! success
    Votre serveur DHCP KEA est maintenant opérationnel.
