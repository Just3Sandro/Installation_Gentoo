# **🐧 Installation de Gentoo Linux – Partie 1 : Préparation du système**

---

## **📦 1\. Partitionnement du disque (GPT)**

je choisis **GPT (GUID Partition Table)** car :

* c’est le standard moderne.

* il est requis pour le démarrage UEFI,

* il est plus fiable que MBR (DOS).

Partitionnement réalisé avec `fdisk /dev/sda`, puis création de deux partitions :

* `/dev/sda1` : 512 Mo – utilisée pour la partition EFI

* `/dev/sda2` : le reste – utilisée pour la racine `/`

---

## **💽 2\. Formatage des partitions**

### **➤ Formatage de la partition EFI :**

`mkfs.fat -F32 /dev/sda1` -> Si une variante FAT n'est pas utilisée pour l'ESP, le micrologiciel UEFI du système n'est pas sûr de trouver le chargeur de démarrage (ou le noyau Linux) et ne sera probablement pas en mesure de démarrer le système !

👉 La partition EFI doit obligatoirement être en **FAT32**, car c’est le seul format lisible nativement par le firmware UEFI pour charger le bootloader (`.efi`).

### **➤ Formatage de la partition racine :**

`mkfs.ext4 /dev/sda2`

👉 On utilise **ext4**, un système de fichiers stable et performant, pour y installer Gentoo.

---

## **📁 3\. Montage des partitions**

### **➤ Montage de la racine :**

`mount /dev/sda2 /mnt/gentoo`

👉 On prépare l’environnement d’installation : `/mnt/gentoo` va devenir la base temporaire du futur système Gentoo.

### **➤ Création et montage du point `/boot/efi` :**

`mkdir -p /mnt/gentoo/boot/efi`  
`mount /dev/sda1 /mnt/gentoo/boot/efi`

👉 Préparation de l’arborescence pour pouvoir plus tard installer GRUB (le bootloader EFI) à l’intérieur de `/boot/efi`.

---

## **🧠 4\. Création et activation du fichier swap**

### **➤ Création d’un fichier de 2 Go :**

`dd if=/dev/zero of=/mnt/gentoo/swap bs=1G count=2`  
`chmod 600 /mnt/gentoo/swap`

👉 `dd` génère un fichier vide de 2 Go rempli de zéros.  
 👉 `chmod 600` protège le fichier (lecture/écriture autorisée uniquement à root).

### **➤ Formatage et activation :**

`mkswap /mnt/gentoo/swap`  
`swapon /mnt/gentoo/swap`

👉 Le fichier devient un espace de **swap** : utilisé comme **mémoire virtuelle** si la RAM est saturée, ou pendant de longues compilations (comme dans Gentoo).

## 🔁 5\. Montage des systèmes essentiels (proc, sys, dev, run)

`mount --types proc /proc /mnt/gentoo/proc`

`mount --rbind /sys /mnt/gentoo/sys`

`mount --make-rslave /mnt/gentoo/sys`

`mount --rbind /dev /mnt/gentoo/dev`

`mount --make-rslave /mnt/gentoo/dev`

`mount --bind /run /mnt/gentoo/run`

👉 Ces commandes permettent au système chrooté d’accéder au matériel, aux périphériques, et à l’environnement du liveCD.  
 Sans ces montages, il serait impossible d’installer le système ou GRUB proprement.

---

## 🛠️ 6\. Entrée dans l’environnement chroot

`chroot /mnt/gentoo /bin/bash`

`source /etc/profile`

`export PS1="(chroot) ${PS1}"`

👉 Le chroot permet de "changer de racine" : on travaille **comme si Gentoo était déjà installé**, avec son propre `/etc`, `/usr`, etc.  
 C’est une étape clé avant toute installation système réelle.

---

## 🔧 7\. Sélection du profil système

`eselect profile list`

`eselect profile set X`

👉 Gentoo propose plusieurs profils préconfigurés (serveur, desktop, hardened, systemd…).  
 Ici, on a choisi le profil **`default/linux/amd64/17.1`**, qui est **neutre, minimal et parfait pour apprendre**.  
 (Note : un passage futur au profil `hardened` est prévu pour durcir la sécurité système.)

---

## **💻 8\. Installation et configuration de GRUB en mode UEFI**

### **➤ Activation du support UEFI dans GRUB**

`mkdir -p /etc/portage/env`

`echo 'GRUB_PLATFORMS="efi-64"' > /etc/portage/env/grub-platforms.conf`

`echo 'sys-boot/grub grub-platforms.conf' >> /etc/portage/package.env`

`emerge --ask --oneshot sys-boot/grub`

👉 Gentoo ne compile que ce qu’on lui demande. Il faut explicitement activer le support **`efi-64`** pour que GRUB puisse s’installer sur une machine UEFI.

---

### **➤ Installation de GRUB dans la partition EFI**

`grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=gentoo`

`grub-mkconfig -o /boot/grub/grub.cfg`

👉 Ces commandes :

* Installent GRUB dans le bon format pour l’UEFI

* Génèrent automatiquement le fichier de configuration `grub.cfg` avec détection du noyau installé

---

## **🧩 9\. Installation des sources du noyau**

`emerge --ask sys-kernel/gentoo-sources`

`cd /usr/src/linux`

👉 On installe ici les **sources du noyau Linux** qu’on va compiler à la main, pour les adapter au matériel et besoins spécifiques (réseau, système de fichiers, sécurité, etc.)

---

### **(Optionnel) 10\. Préparation du système pour intégration automatique du noyau**

`echo "sys-kernel/installkernel dracut grub" >> /etc/portage/package.use/installkernel`

`emerge --ask sys-kernel/installkernel`

👉 Cela permet, après compilation manuelle du noyau, de le copier automatiquement dans `/boot`, d’ajouter un initramfs via Dracut, et de le rendre visible dans GRUB.

## 🛠️ 11\. Compilation manuelle du noyau Linux

make menuconfig  
make \-j$(nproc)  
make modules\_install  
make install

✅ **Explication :**

* `make menuconfig` : interface de configuration du noyau. j’ai ajusté les paramètres selon mon CPU (Intel i9-9900K), activé LZ4 pour la compression et ajusté le buffer log kernel. J’ai aussi changé certaines choses sur les drivers graphiques car installation sur VM

* `make -j$(nproc)` : compilation optimisée avec tous les cœurs (ici, 16 threads pour le i9).

* `make modules_install` : installation des modules noyau.

* `make install` : copie du noyau dans `/boot`, mais **initialement mal monté**, donc il ne créait qu'un fichier `/boot/vmlinuz`.

🚧 **Erreur rencontrée :** Le fichier `/boot/vmlinuz-6.12.31-gentoo` attendu par GRUB n'était pas là car `/boot` n'était pas monté lors de la première compilation.

🔄 **Solution :**

* Monter `/boot` correctement

* Renommer le fichier :

mv /boot/vmlinuz /boot/vmlinuz-6.12.31-gentoo

🏠 12\. Génération de l'initramfs avec Dracut

dracut \--kver 6.12.31-gentoo

✅ Crée un fichier `initramfs-6.12.31-gentoo.img` utilisé par le noyau pour démarrer.

🌐 13\. Mise à jour de GRUB avec le bon noyau

grub-mkconfig \-o /boot/grub/grub.cfg  
grep linux /boot/grub/grub.cfg

✅ GRUB détecte maintenant `/boot/vmlinuz-6.12.31-gentoo` et l'ajoute correctement dans le menu de démarrage.

❌ **Erreurs rencontrés :**

* **GRUB ne voyait rien** tant que le noyau n'était pas nommé correctement

* **Pas de `initramfs`** car `dracut` non utilisé au départ

* **Montage de `/boot` oublié** → très fréquent

🔐 14\. Sortie du chroot et reboot

exit  
umount \-lR /mnt/gentoo  
reboot

🔹 En résumé :

* On a appris la logique du partitionnement UEFI, les bases du noyau Linux, la gestion d'un environnement chroot, et à lire les erreurs pour avancer.

* Gentoo, c’est exigeant, mais incroyablement formateur.

## Final

Malheureusement (je pense que c’est à cause de la VM), l'installation plante au début du chargement du disque mémoire initial. J’ai donc utilisé le noyau Gentoo par défaut, ce qui fait que toutes les modifications effectuées dans le menu `make menuconfig` n'ont pas été prises en compte, mais j’ai tout de même pu me familiariser avec son fonctionnement.  
![][image1]

Source :  
[https://wiki.gentoo.org/wiki/Handbook:AMD64](https://wiki.gentoo.org/wiki/Handbook:AMD64) (la doc de gentoo, incroyable \!)   
[https://www.linuxtricks.fr/wiki/installer-gentoo-facilement-systemd](https://www.linuxtricks.fr/wiki/installer-gentoo-facilement-systemd) 

[https://www.youtube.com/watch?v=xWJFZAk1jNw](https://www.youtube.com/watch?v=xWJFZAk1jNw)   
[https://www.youtube.com/watch?v=KE1MPz50Yo0](https://www.youtube.com/watch?v=KE1MPz50Yo0)  
[https://www.youtube.com/watch?v=NVWVHiLx1sU\&t=1656s](https://www.youtube.com/watch?v=NVWVHiLx1sU&t=1656s) (bonne vidéo pour la configuration du noyau \!)

[image1]: <data:image/png;base64,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>
