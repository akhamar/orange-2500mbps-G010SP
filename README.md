# Orange 2500Mbps

## Préconditions

La cible de cette explication est pour le materiel / software suivant

- Orange en offre `Livebox Up Fibre`
- Un convertisseur Fibre vers RJ45 `MC220L Tp-Link`
- Un ONU `G-010S-P`
- Une carte SFP+ `N20KJ [Broadcom 57810 S]`
- Un routeur sous `OpnSense [21.x]`




## Passage du G-010S-P en Carlitoxx V1

### SSH vers G-010S-P
> IP: 192.168.1.10
> 
> login: ONTUSER
> 
> password: SUGAR2A041

Dans un premier temps, il faut utiliser l'ONU avec le MC220L, brancher le MC220L directement sur un PC en fixant une ip static de type 192.168.1.X (différente de 192.168.1.10).

Une fois cette étape faite, il suffit de ping l'ONU via `ping -t -w 30 192.168.1.10`.

L'ONU devrait répondre au ping (une fois celui-ci démarré).

### Backup l'ONU G-010S-P avant flash

Backup via dd

> ssh ONTUSER@192.168.1.10

```
dd if=/dev/mtd0 of=/tmp/mtd0.backup
dd if=/dev/mtd1 of=/tmp/mtd1.backup
dd if=/dev/mtd2 of=/tmp/mtd2.backup
dd if=/dev/mtd3 of=/tmp/mtd3.backup
dd if=/dev/mtd4 of=/tmp/mtd4.backup
dd if=/dev/mtd5 of=/tmp/mtd5.backup
```

Transfert des backup

```
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd0.backup mtd0.backup
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd1.backup mtd1.backup
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd2.backup mtd2.backup
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd3.backup mtd3.backup
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd4.backup mtd4.backup
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd5.backup mtd5.backup
```

### Flash en Carlitoxx V1

Transfert des images mtd2.bin et mtd5.bin

Recupérez les images de la carlitoxx v1 : [Carlitoxx v1](CarlitoxV1.zip)

```
scp -o KexAlgorithms=diffie-hellman-group1-sha1 mtd2.bin ONTUSER@192.168.1.10:/tmp/
scp -o KexAlgorithms=diffie-hellman-group1-sha1 mtd5.bin ONTUSER@192.168.1.10:/tmp/
```

SSH sur l'ONU

> ssh ONTUSER@192.168.1.10

```
mtd -e image0 write /tmp/mtd2.bin image0
mtd -e linux write /tmp/mtd5.bin linux
```
`*** sur un G-010S-P image0 correspond à mtd2 et linux mtd3 ***`

Il faut ensuite set les valeurs avant un reboot

```
fw_setenv ont_serial XXXXXXXXX
fw_setenv target oem-generic
fw_setenv committed_image 0
reboot
```

> Où ont_serial [XXXXXXXXX] correspond au numero de série de l'ONT Orange
> 
> Pour trouver ses infos, se référer à l'image en bas




## Configuration de l'ONU

### Configuration de Carlitoxx v1

Une fois l'ONU redémarré via un test `ping -t -w 30 192.168.1.10`

Aller sur l'interface web de celui-ci `http://192.168.1.10`

Lors de votre première connexion, il vous sera demandé d'entrer un compte / password. Ces identifiants seront ceux utilisé en SSH et en HTTP par la suite.

Il est possible que l'utilisateur soit root par defaut en SSH.

SSH sur l'ONU

> ssh YOUR_NEW_USER@192.168.1.10

`vi /etc/init.d/sys.sh`
```
Editez la partie oem-generic qui correspond au paramétrage fait à l'étape précédente (fw_setenv target oem-generic)
Les valeurs à saisir sont indiqué dans l'image.
```

Il faudra aussi configurer un ensemble de variable d'env (cf. image)

```
fw_setenv target oem-generic
fw_setenv ont_serial XXXXXXXXXX
fw_setenv image0_version XXXXXXXXXX
fw_setenv image1_version XXXXXXXXXX
fw_setenv sgmii_mode 5
```

![Alt text](LhkrqUL38M.png?raw=true "Config ONU")

> Reboot

### Test de l'ONU dans le MC220L

Il est maintenant temps de vérifier que la configuration de l'ONU permet de se connecté à l'OLT, d'obtenir les VLAN et de pouvoir obtenir un traffic montant et descendant.

SSH sur l'ONU

> ssh YOUR_NEW_USER@192.168.1.10

---

`onu ploamsg`

> curr_state doit être a 5

---

`onu gtcsng`

> Votre ont_serial doit apparaitre ici

---

`gtop` puis `c` et `v`

ou

`gtop` puis `c` et `y`

> Vous devriez visualiser les VLAN typique orange (832, ...)

Si vous êtes arrivé jusqu'ici, normalement votre ONU est correctement configuré.




## Configuration de la carte SFP+

### Utilisation de l'outil eDiag

Recupérez l'outil eDiag : [eDiag](NX2_Ev.zip)

Créez une clef USB bootable en FreeDos (via rufus par exemple), ajoutez l'ensemble des fichiers contenu dans `NX2_Ev.zip` à la racine de la clef USB, puis bootez sur celle-ci.

Lancez l'executable via `ediag.exe -b10eng`

Vous allez pouvoir configurer chacune des deux interface (device 1 et device 2)

```
device 1
nvm cfg
6
35=70
36=70
56=6
59=6
save
exit
```
> Il se peut que 6 ne soit pas [Links] mais [Feature], cela ne change pas la manière de faire. Feature étant juste un menu plus complet.

> Faire de même avec le device 2 dans le cas ou vous voulez aussi configurer cette interface pour du 2.5Gbps par default. Plus simple pour être sur que peut importe l'interface utilisé vous serez en 2500Mbps.




## Configuration du routeur OpnSense

### Driver BXE

Recupérez le driver : [bxe](if_bxe.ko.zip)

Transférez le driver sur votre routeur OpnSense, puis remplacer le driver d'origine.

```
mv /boot/kernel/if_bxe.ko /boot/kernel/if_bxe.ko.bak
mv /SCP_UPLOAD PATH/if_bxe.ko /boot/kernel/if_bxe.ko
chmod 555 /boot/kernel/if_bxe.ko
```

### Modification du driver BXE via patch (optionnel)

Recupérez le driver : [patch](bxe_8727_warpcore_2_5g.patch)

Et tentez de l'appliquer ou de le patcher manuellement vous même.

Emplacement pour patcher `/usr/src/sys/dev/bxe`

Emplacement pour compiler et créer le .ko `/usr/src/sys/modules/bxe`
```
cd /usr/src/sys/modules/bxe
make
mv /boot/kernel/if_bxe.ko /boot/kernel/if_bxe.ko.bak
cp if_bxe.ko /boot/kernel/if_bxe.ko
```

### Paramétrage

`vi /boot/loader.conf.local`

```
autoboot_delay=30

hw.pci.honor_msi_blacklist="0"

hw.bxe.interrupt_mode="1"
net.inet.tcp.tso="0"
if_bxe_load="YES"
```

Il est possible que la ligne `autoboot_delay=30` soit nécéssaire. En effet l'ONU étant assez long a boot, un routeur démarrant assez rapidement peut poser problème. On peut donc virtuellement ralentir le boot, pour laisser un peu plus de temps a l'ONU pour boot.

Il faut aussi désactivé les différentes accélérations matérielles

> Disable Hardware CRC (hardware checksum offload), Hardware TSO (hardware tcp segmentation offload), Hardware LSO (hardware large receive offload) and VLAN Hardware Filtering (pfsense and opnsense)


# Annexes

## Liens d'info bien formé et propre

[https://www.dslreports.com/forum/r32230041-Internet-Bypassing-the-HH3K-up-to-2-5Gbps-using-a-BCM57810S-NIC](https://www.dslreports.com/forum/r32230041-Internet-Bypassing-the-HH3K-up-to-2-5Gbps-using-a-BCM57810S-NIC)

[https://github.com/Berzerker/google-fiber-2gbps-bypass](https://github.com/Berzerker/google-fiber-2gbps-bypass)

[https://www.rapidtables.com/convert/number/ascii-to-hex.html](https://www.rapidtables.com/convert/number/ascii-to-hex.html)


## Liens d'info bordélique

[https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/msg831004/#msg831004](https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/msg831004/#msg831004)

[https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/msg845067/#msg845067](https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/msg845067/#msg845067)

[https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/msg848578/#msg848578](https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/msg848578/#msg848578)


## Autres

[https://forum.openwrt.org/t/support-ma5671a-sfp-gpon/48042/33](https://forum.openwrt.org/t/support-ma5671a-sfp-gpon/48042/33)

[https://github.com/hwti/G-010S-A](https://github.com/hwti/G-010S-A)

[https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/](https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/)



## Récupération de la config d'authentification
![Alt text](LhkrqUL38M.png?raw=true "Config ONU")


## Il est possible qu'il soit nécessaire de set le lien `sgmii_mode` en 2.5Gbps pour que l'ONU fonctionne correctement

`fw_setenv sgmii_mode 5`

Check link speed `onu lanpsg` or `onu lanpsg 0`


## Fan warning on N20KJ [Broadcom 57810 S]

> In the ediag, I turned off the fan warning (83=1) to bypass the fan and now my board/SFP are running cooler.

Pour supprimer le fan fail (dans le cas ou le ventilateur est supprimé => cas ou le boitier est bien ventilé ou autre), il suffit avec ediag de configurer le champs qui active le fan fail/error.

```
ediag -b10eng
nvm cfg
option 4 (board i/o)
83=1 (disabled)
save
exit
```
