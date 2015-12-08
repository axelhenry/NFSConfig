# Mise en place d'un serveur nfs dans une machine virtuelle

Mon installation multimédia se compose d'un raspberry pi connecté à une télé, obtenant son contenu d'un ordinateur fixe ne restant pas allumé en permanence. La problématique ici était de permettre le montage et le démontage automatique des partages sur le raspberry sans occasionner de freeze de la commande mount, qui oblige à redémarrer le raspberry à chaque fois que l'on veut remonter les partages.
Pour se faire, la mise en place d'un serveur nfs semble être une solution simple à mettre en oeuvre.

## Installation d'un serveur nfs sous Windows 7 64 bits.

Pour mettre en place un serveur nfs, quelques outils existent :
- [Windows Service for Unix](https://www.microsoft.com/en-us/download/details.aspx?id=274), développé par Microsoft, gratuit, disponible seulement pour les systèmes d'exploitation __32bits__ (la déception).
- [WinNFSd](https://github.com/winnfsd/winnfsd/releases), opensource, semblait prometteur, mais je n'ai pas réussi à exporter plusieurs dossiers avec.
- [haneWIN NFS Server](http://www.hanewin.net/nfs-e.htm), shareware.
- [Allegro NFS](http://nfsforwindows.com/), shareware.

Aucune de ces solutions n'ayant donné satisfaction, je me suis tourné vers la solution de la virtualisation.

### Installation d'un serveur nfs virtualisé.

Les procédures qui suivent ont été réalisées sous VMware Player 12 et VMware Workstation 12, mais doivent être réalisables avec des solutions telles que Virtualbox.

1.  Installer VMware (Player ou Workstation)

1. Télécharger une image .iso d'une distribution Linux (par habitude, la distribution [Archlinux](https://www.archlinux.org/download/) sera utilisée ici, mais beaucoup d'autres peuvent faire l'affaire).

1. Créer une nouvelle machine virtuelle (noyau 3.x, 32 bits, 128mo de ram et 2go d'espace disque devraient être suffisants.).

1. Passer l'adaptateur réseau de votre machine virtuelle en mode Bridged.

1. Installer votre distribution (se référer aux tutos de votre distribution), pour économiser de la place vous pouvez vous dispenser d'installer une interface graphique.

1. Installer et configurer [sudo](https://wiki.archlinux.org/index.php/Sudo)

1. Si vous avez installé Archlinux, ne vous arrêtez pas en si bon chemin et installez [yaourt](https://wiki.archlinux.fr/Yaourt) à la place de pacman.

1. Installer les paquets nécessaires
  - nfs-utils pour la prise en charge du système de fichier nfs.
  - open-vm-tools et open-vm-tools-dkms pour la prise en charge des fonctionnalités liées au système de virtualisation (montage des dossiers partagés).
  - unfs3 pour avoir un serveur nfs qui puisse fonctionner en espace utilisateur et exporter les dossiers partagés (je sais, c'est du chinois).

   ```sh
yaourt -S nfs-utils open-vm-tools open-vm-tools-dkms
yaourt -S unfs3 --force
```

1. Charger le [module](https://wiki.archlinux.org/index.php/VMware/Installing_Arch_as_a_guest#Shared_Folders) :

   ```sh
   sudo modprobe vmhgfs
   ```

1. Activer le [module](https://wiki.archlinux.org/index.php/VMware/Installing_Arch_as_a_guest#Shared_Folders) au démarrage :

   ```sh
   sudo nano /etc/mkinitcpio.conf
   ```

   Ajouter vmhgfs à vos modules :
   ```
   ...
   MODULES="... vmhgfs"
   ...
   ```
   Mettre à jour votre ramdisk :
   ```sh
   sudo mkinitcpio -p linux
   ```


1. Eteindre votre machine virtuelle, activer les dossiers partagés et ajouter les dossiers que vous voulez mettre à disposition de votre machine virtuelle.

1. Redémarrer votre machine virtuelle.

1. Créer les points de montage de vos dossiers partagés (mes dossiers partagés sont ici __HD__, __Series__ et __lowres__).

   ```sh
sudo mkdir /mnt/hd
sudo mkdir /mnt/series
sudo mkdir /mnt/lowres
```

1. Ouvrir le fichier /etc/fstab

   ```sh
sudo nano /etc/fstab
```

   et ajouter ces lignes :

   ```
.host:/HD /mnt/hd vmhgfs defaults 0 0
.host:/Series /mnt/series vmhgfs defaults 0 0
.host:/lowres /mnt/lowres vmhgfs defaults 0 0
```
   Sauvegarder en faisant ctrl + x.

1. Redémarrer votre machine virtuelle, et exécuter :

   ```sh
ls /mnt/hd
```
   Vous devriez voir le contenu de votre dossier partagé.

1. Créer les points de montage nfs.

   ```sh
sudo mkdir -p /srv/nfs
sudo mkdir -p /srv/nfs/hd
sudo mkdir -p /srv/nfs/series
sudo mkdir -p /srv/nfs/lowres
```

1. Ouvrir le fichier /etc/fstab

   ```sh
sudo nano /etc/fstab
```
   et ajouter ces lignes :

   ```
/mnt/hd /srv/nfs/hd none bind 0 0
/mnt/series /srv/nfs/series none bind 0 0
/mnt/lowres /srv/nfs/lowres none bind 0 0
```
   Sauvegarder en faisant ctrl + x

1. Ouvrir le fichier /etc/exports

   ```sh
sudo nano /etc/exports
```

   et ajouter ces lignes :

   ```
/srv/nfs/series
/srv/nfs/hd
/srv/nfs/lowres
```
   Sauvegarder via ctrl + x

1. Ouvrir le fichier /etc/conf.d/nfs-common.conf

   ```sh
nano /etc/conf.d/nfs-common.conf
```
  - Modifier la ligne STATD_OPTS:

     ```
  STATD_OPTS="-p 32765 -o 32766 -T 32803"
  ```

1. Ouvrir le fichier /etc/conf.d/unfsd.conf (le chemin peut varier en fonction de la distribution employée)

   ```sh
sudo nano /etc/conf.d/unfsd.conf
```
  - Commenter la ligne (mettre # devant la ligne):

     ```
  PORT_UNPRIVILEGED="-u"
  #PORT_UNPRIVILEGED="-u"
```

 - Vérifier que les lignes PORT_NFS and PORT_MOUNT ne sont pas commentées:

    ```
 PORT_NFS="2049"
 PORT_MOUNT="2049"
 ```
 Sauvegarder via ctrl + x

1. Ouvrir les ports nécessaires dans le pare-feu :

   ```sh
sudo cp /etc/iptables/empty.rules /etc/iptables/iptables.rules
sudo iptables -A INPUT -p tcp -m tcp --dport 111 -j ACCEPT
sudo iptables -A INPUT -p udp -m udp --dport 111 -j ACCEPT
sudo iptables -A INPUT -p tcp -m tcp --dport 2049 -j ACCEPT
sudo iptables -A INPUT -p udp -m udp --dport 2049 -j ACCEPT
sudo iptables -A INPUT -p tcp -m tcp --dport 20048 -j ACCEPT
sudo iptables -A INPUT -p udp -m udp --dport 20048 -j ACCEPT
sudo iptables -A INPUT -p tcp -m tcp --dport 32765 -j ACCEPT
sudo iptables -A INPUT -p udp -m udp --dport 32765 -j ACCEPT
sudo iptables -A INPUT -p tcp -m tcp --dport 32803 -j ACCEPT
sudo iptables -A INPUT -p udp -m udp --dport 32803 -j ACCEPT
sudo iptables-save > /etc/iptables/iptables.rules
```

1. Activer le service unfsd :

   ```sh
sudo systemctl enable unfsd.service
```

1. Redémarrer votre machine virtuelle.

1. Sur un ordinateur sous linux, taper:

   ```sh
showmount -e <server_name>
```
Vous devriez voir la liste de vos partages nfs.

### Démarrage automatique de notre machine virtuelle.

1. Maintenant que notre serveur nfs fonctionne, notre fainéantise nous pousse à vouloir le lancer systématiquement au démarrage de notre ordinateur. Pour cela, installer VMware Workstation.

1. Aller dans

   ```
X:\Program Files(x86)\VMware\VMware VIX\
```

1. Créer un raccourci vers vmrun.exe.

1. Clic droit sur le raccourci nouvellement créé, propriétés.

1. Dans la cible, vous devriez avoir :

   ```
"X:\Program Files(x86)\VMware\VMware VIX\"
```
Modifier la cible afin d'avoir :

   ```
"X:\Program Files(x86)\VMware\VMware VIX\vmrun.exe" -T ws start "Y:\Chemin\Vers\Votre\Machine\Virtuelle.vmx"
```
Appuyer sur ok.

1. Pour lancer ce programme au démarrage de votre ordinatuer, copier le raccourci vers:

   ```
X:\Utilisateurs\xxxxx\AppData\Roaming\Microsoft\Windows\Menu Démarrer\Programmes\Démarrage\
```

1. Well done, votre machine virtuelle devrait maintenant se lancer au démarrage de votre ordinateur


## Configuration de notre raspberry pour le montage et le démontage automatique de nos partages nfs.

Très fortement inspiré de ce [guide](https://wiki.archlinux.org/index.php/NFS#Automatic_mount_handling), avec quelques corrections.

1. Installer nfs-utils

   ```sh
yaourt -S nfs-utils
```

1. Créer les points de montage de nos partages.

   ```sh
sudo mkdir /media/hd
sudo mkdir /media/lowres
sudo mkdir /media/series
```

1. Ouvrir /etc/fstab

   ```sh
sudo nano /etc/fstab
```
Ajouter ces lignes:

   ```
<server_name>:/srv/nfs/hd /mnt/hd nfs noauto,noatime,rsize=32768,wsize=32768,vers=3 0 0
<server_name>:/srv/nfs/lowres /mnt/lowres nfs noauto,noatime,rsize=32768,wsize=32768,vers=3 0 0
<server_name>:/srv/nfs/series /mnt/series nfs noauto,noatime,rsize=32768,wsize=32768,vers=3 0 0
```
Sauvegarder via ctrl + x

1. Télécharger le script auto_share.sh :

   ```sh
sudo wget -P /usr/local/bin/ http://github.com/axelhenry/NFSConfig/raw/master/files/auto_share.sh
```

1. Rendre ce script exécutable.

   ```sh
sudo chmod +x /usr/local/bin/auto_share.sh
```

1. Télécharger le timer auto_share.timer :

   ```sh
sudo wget -P /etc/systemd/system/ http://github.com/axelhenry/NFSConfig/raw/master/files/auto_share.timer
```

1. Télécharger le service auto_share.service :

   ```sh
sudo wget -P /etc/systemd/system/ http://github.com/axelhenry/NFSConfig/raw/master/files/auto_share.service
```

1. Lancer le service au démarrage.

   ```sh
sudo systemctl enable auto_share.timer
```

1. Redémarrer votre raspberry et faire des tests, les partages devraient apparaître quand votre serveur physique est en ligne.

1. GG
