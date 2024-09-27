
# Documentation : Connexion Automatique à un Réseau Wi-Fi et Configuration d'un Pont Réseau sur Proxmox

## Introduction
Cette documentation couvre les étapes pour configurer un serveur Proxmox afin qu'il se connecte automatiquement à un réseau Wi-Fi au démarrage et pour configurer un pont réseau (`vmbr0`) dans le même réseau que l'interface Wi-Fi (`wlp3s0`). Ce guide inclut la création d'un script d'automatisation des commandes nécessaires et l'intégration de ce script dans le processus de démarrage du système à l'aide de deux méthodes : `/etc/rc.local` et `systemd`.

## 1. Prérequis
Avant de commencer, assurez-vous que :
- Votre serveur Proxmox est équipé d'une carte Wi-Fi compatible.
- Vous avez accès à votre réseau Wi-Fi (SSID et mot de passe).
- Les outils nécessaires, comme `wpa_supplicant` et `dhclient`, sont installés sur votre système.
```bash
apt update
apt install wpasupplicant
```
- A partir de maintenant, Ethernet n'est plus nécessaire !
## 2. Connexion Wi-Fi manuelle

### 2.1. Configurer `wpa_supplicant`
`wpa_supplicant` est utilisé pour gérer les connexions Wi-Fi sous Linux. Vous devez créer un fichier de configuration pour `wpa_supplicant` qui contient les détails de votre réseau Wi-Fi.

1. Créer le fichier de configuration :
    ```bash
    sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
    ```
2. Ajouter la configuration du réseau :
    ```bash
    network={
        ssid="votre_ssid"
        psk="votre_mot_de_passe"
    }
    ```
3. Enregistrer et fermé le fichier avec `Ctrl+O` pour enregistrer `Ctrl+X`.

### 2.2. Connexion au réseau Wi-Fi
Pour connecter manuellement votre serveur au réseau Wi-Fi, utilisez les commandes suivante :

1. Démarrer `wpa_supplicant` :
    ```bash
    sudo wpa_supplicant -B -i wlp3s0 -c /etc/wpa_supplicant/wpa_supplicant.conf
    ```
    > Remarque : Remplacez `wlp3s0` par le nom de votre interface Wi-Fi, que vous pouvez obtenir avec la commande `ip a`.

2. Obtenir une adresse IP :
    ```bash
    sudo dhclient wlp3s0
    ```

## 3. Automatisation au démarrage
Pour que le serveur se connecte automatiquement au Wi-Fi au démarrage, nous allons créer un script shell qui exécutera les commandes ci-dessus, puis configurer ce script pour qu'il s'exécute automatiquement.

### 3.1. Création du script shell

1. Créer le script :
    ```bash
    sudo nano /usr/local/bin/connect_wifi.sh
    ```
2. Ajouter les commandes au script :
    ```bash
    #!/bin/bash
    sudo wpa_supplicant -B -i wlp3s0 -c /etc/wpa_supplicant/wpa_supplicant.conf
    sudo dhclient wlp3s0
    ```
3. Rendre le script exécutable :
    ```bash
    sudo chmod +x /usr/local/bin/connect_wifi.sh
    ```

### 3.2. Intégration du script au démarrage
Il existe deux principales méthodes pour exécuter ce script automatiquement au démarrage de votre serveur : en utilisant `/etc/rc.local` ou en créant un service `systemd`.

#### Option 1 : Utiliser `/etc/rc.local`

1. Modifier `/etc/rc.local` :
    - Si le fichier `/etc/rc.local` n'existe pas, créez-le :
    ```bash
    sudo nano /etc/rc.local
    ```
2. Ajouter le script au fichier :
    Ajoutez la ligne suivante avant `exit 0` :
    ```bash
    /usr/local/bin/connect_wifi.sh
    ```
3. Rendre `/etc/rc.local` exécutable (si ce n’est pas déjà fait) :
    ```bash
    sudo chmod +x /etc/rc.local
    ```

#### Option 2 : Créer un service `systemd`

1. Créer un fichier de service `systemd` :
    ```bash
    sudo nano /etc/systemd/system/connect_wifi.service
    ```
2. Ajouter la configuration suivante au fichier :
    ```bash
    [Unit]
    Description=Connect to Wi-Fi at startup
    After=network.target

    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/connect_wifi.sh
    RemainAfterExit=true

    [Install]
    WantedBy=multi-user.target
    ```
3. Activer le service :
    Pour que le service soit exécuté au démarrage :
    ```bash
    sudo systemctl enable connect_wifi.service
    ```
4. Tester le service :
    Pour démarrer le service manuellement et vérifier son fonctionnement :
    ```bash
    sudo systemctl start connect_wifi.service
    ```

## 4. Configuration du Pont Réseau (`vmbr0`) avec l'Interface Wi-Fi (actuellement en test)
Proxmox utilise `vmbr0` pour connecter les machines virtuelles au réseau. Si vous souhaitez que `vmbr0` soit dans le même réseau que `wlp3s0` (votre interface Wi-Fi), vous devez configurer le pont réseau pour utiliser l'interface Wi-Fi.

### 4.1. Configurer `/etc/network/interfaces`

1. Modifier `/etc/network/interfaces` :
    ```bash
    sudo nano /etc/network/interfaces
    ```
2. Ajouter ou modifier la configuration suivante :
    ```bash
    auto wlp3s0
    iface wlp3s0 inet manual

    auto vmbr0
    iface vmbr0 inet dhcp
        bridge_ports wlp3s0
        bridge_stp off
        bridge_fd 0
    ```
    > Remarque : Si la carte Wi-Fi ne supporte pas le mode pont, cette configuration pourrait ne pas fonctionner. Dans ce cas, vous devrez utiliser un autre moyen, comme la configuration de NAT avec `iptables`.

3. Redémarrer les interfaces réseau :
    ```bash
    sudo systemctl restart networking
    ```

## 5. Résumé
Vous avez configuré votre serveur Proxmox pour se connecter automatiquement à un réseau Wi-Fi au démarrage en utilisant un script shell, et vous avez intégré ce script dans le processus de démarrage en utilisant soit `/etc/rc.local`, soit un service `systemd`. De plus, vous avez configuré le pont réseau `vmbr0` pour fonctionner dans le même réseau que `wlp3s0`.

## 6. Dépannage

- **Vérifiez le statut du service** : Si vous utilisez `systemd`, vous pouvez vérifier le statut du service pour diagnostiquer des problèmes :
    ```bash
    sudo systemctl status connect_wifi.service
    ```
- **Logs de `wpa_supplicant`** : Les erreurs liées à `wpa_supplicant` peuvent être examinées dans les logs système :
    ```bash
    sudo journalctl -u wpa_supplicant
    ```
- **Redémarrage manuel des services** : Si quelque chose ne fonctionne pas comme prévu, redémarrez manuellement les services ou le serveur pour voir si le problème persiste.

---

Cette documentation devrait vous aider à configurer et automatiser la connexion Wi-Fi sur votre serveur Proxmox, ainsi qu'à configurer un pont réseau pour vos machines virtuelles. Si vous rencontrez des difficultés supplémentaires, n’hésitez pas à les revisiter ou à demander de l'aide supplémentaire.
