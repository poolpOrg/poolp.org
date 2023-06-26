---
title: "Installer OpenBSD sur une dedibox Classic+ Gen2"
date: 2013-05-08 14:23:43
category: OpenBSD
authors:
 - Gilles Chehade
categories:
 - technology
---

Yop,

Les services de poolp.org sont hébergés sur plusieurs serveurs.

Le serveur principal à été basculé d'un petit hébergeur US "OpenBSD-friendly" vers Online.net depuis 3 ans maintenant. J'ai rusé à l'époque pour installer OpenBSD sur la Dedibox "DC" en utilisant une vieille version de Yaifo et en faisant des upgrades successives du système d'exploitation. Mis à part 3 petits downtimes dont 1 de ma responsabilité, je n'ai jamais eu à me plaindre de leurs services et de leur matériel.

Le serveur secondaire, qui ne fournit que de l'espace disque pour les backups et un service de DNS et de SMTP secondaire en cas de panne du serveur primaire, est actuellement un serveur Kimsufi de chez OVH sur lequel j'ai installé OpenBSD via le vKVM. Le serveur est sous-utilisé, recevant un trafic anecdotique en dehors des backups journaliers en provenance du serveur primaire, mais je suis globalement content du service qui n'a pas subi de downtime depuis plus d'un an en ce qui me concerne.

Des fois, pour les besoins d'un projet, je prends un serveur supplémentaire pour une durée indéterminée et je le laisse expirer à la fin du projet, ou bien je bascule le serveur secondaire dessus si à prix équivalent le matériel est plus confortable.

Ce week-end, je me suis rendu compte de la disponibilité des nouveaux serveurs Dedibox Classic+ Gen2 qui avaient l'air pas mal sexy. J'allais justement renouveller un serveur un peu moins confortable au même tarif, j'ai donc décidé de prendre un nouveau serveur chez Online et de laisser expirer celui que j'allais renouveller chez OVH.

Le Classic+ Gen2 avait l'air assez sympa et la dispo d'un KVM/IP laissait présager qu'installer OpenBSD serait une partie de plaisir.

Ca, c'était avant d'essayer pour de vrai.

Let's go !

Mon serveur est commandé et je recoit peu après un mail m'indiquant sa disponibilité. Ni une, ni deux, je me loggue et ne vois pas de menu pour avoir accès au KVM, juste un bouton "Install".

Comme il me faudra de toutes facons connaitre un peu le materiel avant d'installer OpenBSD par KVM, je prends le premier OS de la liste et lance une install pour pouvoir recuperer quelques infos du dmesg.

L'install est rapide, il faut attendre un délai d'une heure avant de réussir à se connecter à la machine par SSH. Ca laisse le temps de chercher comment on accède au KVM depuis la console Online.

iDRAC

L'Install de Debian finie, des menus supplémentaires apparaissent dans la console Online, dont le menu iDrac qui permets de lancer une console virtuelle.

Pas de bol, il faut un OS "java"-compliant, je viens de vendre mon laptop et je n'ai rien qui fasse l'affaire. Après moultes galères, je mets la main sur un Windows et je lance la console virtuelle, choisis l'ISO d'OpenBSD et tente un boot.

Pendant le boot, perte de connexion, impossible de relancer la console virtuelle, impossible de rebooter le serveur depuis la console Online ou bien simplement de faire un ssh sur la machine (normal, elle est en plein boot de l'installeur OpenBSD...).

Sur le canal IRC d'#online, on m'indique que l'iDRAC est dans le caniveau, une intervention est planifiee.

Je retente de differentes facons, mais le crash de l'iDRAC est systematique et me fait perdre la main completement sur le serveur sans aucune action possible à part quémander un redémarrage sur IRC et/ou ouvrir un ticket pour planifier une intervention.

A la cinquième tentative, mes reves d'install par KVM/IP s'envolent, on me suggère de faire tourner une VM (ESXi est proposé comme OS), mais je suis pas trop fan donc...

QEMU à la rescousse

Je cherche du coté de Yaifo vu qu'il m'a sauvé la vie la première fois, mais il n'est plus maintenu depuis un bail. J'envisage de le mettre à jour, mais pas trop envie de rester sur le problème 40 ans et l'idée de me retapper des heures d'attentes entre chaque intervention sur mes tentatives de tests échoué m'enchante pas des masses.

Sur IRC, Enjolras (#gcu) me fait savoir qu'il a installé DragonFlyBSD sur un Kimsufi en se servant d'un FreeBSD en mode rescue et d'unionfs pour faire l'install sur le disque physique depuis un QEMU lancé en curses.

Idée séduisante, en rescue sur une Dedibox même pas besoin d'unionfs...

Ci-dessous, voici donc la méthode pour installer le plus simplement possible un OpenBSD sur une Dedibox. Elle devrait marcher pour n'importe quel OS qui supporte le matériel avec un minimum de changements. Merci Enjolras !

Récupérer les infos utiles

Il faut tout d'abord booter sur le Linux installé et récupérer une copie du dmesg.

La raison est toute simple, QEMU va simuler le matériel, l'installation verra un disque et une interface réseau différentes de ceux du serveur physique. Lorsque l'installation sera terminée, si on reboot sur le serveur physique sans avoir fait quelques modifications, le système cherchera une interface et un disque inexistants avec le résultat attendu... un ticket pour une intervention dans quelques heures ;-)

Les infos qui nous intéressent sont éventuellement le DUID pour identifier le disque dans /etc/fstab et la description des interfaces réseau pour retrouver le nom du driver qui les prendra en charge et ainsi pouvoir créer les fichiers de conf réseau.

Sur le Classic+, j'ai décidé de ne pas me servir du DUID vu que c'est sympa mais pas obligatoire et pas d'un intérêt fou dans mon cas. En revanche, un petit grep eth a mis en évidence deux interfaces réseau "Broadcom BCM5716" et un petit coup de man bnx à confirmé que le driver bnx(4) les prends en charge sous OpenBSD. QEMU proposera une interface Realtek prise en charge par le driver re(4), donc ca n'ira pas.

Le disque dur est monté sur /dev/sda, avec OpenBSD dans QEMU il sera visible en tant que /dev/wd0c, et avec OpenBSD sur le serveur physique il sera visible en tant que /dev/sd0c.

De plus, il va nous falloir les infos pour la configuration réseau:

adresse IP
broadcast
netmask
gateway
On peut les récupérer depuis l'interface Online ou depuis ifconfig et route, en ce qui me concerne:

adresse : 88.191.185.XXX
broadcast : 88.191.185.255
netmask : 255.255.255.0
gateway : 88.191.185.1
Il faudra aussi l'adresse d'un serveur de nom pour éviter de passer 4 plombes sur le reverse lookup à la première connexion ssh, personnellement j'ai utilisé le 8.8.8.8 de Google pour mon premier boot puisque j'installe un serveur de nom local par la suite.

Let's Go Pour De Vrai!

Première étape: installer QEMU ;-) % sudo apt-get update % sudo apt-get install qemu

Seconde étape: télécharger l'ISO d'install % wget ftp://ftp.fr.openbsd.org/pub/OpenBSD/5.3/amd64/install53.iso

Troisième étape: lancer l'install. % sudo qemu-system-x86_64 -curses -boot -d -cdrom install53.iso -hda /dev/sda

A ce stade, qemu boot sur l'installer et on fait face a une install tout a fait normale d'OpenBSD. Je le lance en mono-cpu, je prefere une install qui boot sur un kernel monoproc la première fois et faire une install du kernel MP après le premier boot, chacun est libre de faire comme il veut ;-)

Je ne vais pas documenter l'installation, il y a moultes informations à ce sujet, par contre je vais juste faire une petite remarque utile pour vous éviter de perdre du temps inutilement:

en mode "auto", disklabel ne consommera pas tout le disque, il faut faire un découpage custom

Il vaut mieux s'en rendre compte pendant l'install que de se galérer à faire les resize après ... croyez moi.

Installation finie ?

Avant de quitter le mode rescue et de booter sur le serveur physique, quelques petites modifications.

On configure...

... la gateway:

echo 88.191.185.1 > /etc/mygate

... l'interface réseau:

echo inet 88.191.185.XXX 255.255.255.0 88.191.185.255 > /etc/hostname.bnx0

... le resolver:

cat < /etc/resolv.conf
nameserver 8.8.8.8 search file bind EOF

... le fstab (attention à pas se rater):

sed 's/wd0/sd0/g' < /etc/fstab > /tmp/fstab && mv /tmp/fstab /etc/fstab

... et HOP on reboot

reboot

Premier reboot...

Le premier boot est long. Tres long. En fait pas si long que ca, mais l'anxiete de devoir refaire un ticket pour une intervention et attendre 4h pour refaire une tentative donne l'impression que des heures s'ecoulent avant la premiere reponse aux ping ;-)

Une fois ce stade, on doit pouvoir se SSH sur la machine dans un OpenBSD avec un kernel monoproc. Si c'est bon, il ne reste plus qu'à installer le kernel MP:

cp /bsd /bsd.sp
ftp ftp://ftp.fr.openbsd.org/pub/OpenBSD/5.3/amd64/bsd.mp
cp bsd.mp /bsd
sync
reboot

Au reboot on est sur un OpenBSD 5.3 GENERIC#MP \o/

OpenBSD 5.3 (GENERIC.MP) #62: Tue Mar 12 18:21:20 MDT 2013 deraadt@amd64.openbsd.org:/usr/src/sys/arch/amd64/compile/GENERIC.MP real mem = 8557342720 (8160MB) avail mem = 8307052544 (7922MB) mainbus0 at root bios0 at mainbus0: SMBIOS rev. 2.7 @ 0xe6da0 (57 entries) bios0: vendor Dell Inc. version "2.2.3" date 10/25/2012 bios0: Dell Inc. PowerEdge R210 II acpi0 at bios0: rev 2 acpi0: sleep states S0 S4 S5 acpi0: tables DSDT FACP SPMI DMAR ASF! HPET APIC MCFG BOOT SSDT ASPT SSDT SSDT SPCR HEST ERST BERT EINJ acpi0: wakeup devices P0P1(S4) GLAN(S0) EHC1(S4) EHC2(S4) XHC_(S4) PXSX(S4) RP01(S5) PXSX(S4) RP02(S5) PXSX(S4) RP03(S5) PXSX(S4) RP04(S5) PXSX(S4) RP05(S5) PXSX(S4) RP06(S5) PXSX(S4) RP07(S5) PXSX(S4) RP08(S5) PEG0(S5) PEGP(S5) PEG1(S5) PEG2(S5) PEG3(S5) acpitimer0 at acpi0: 3579545 Hz, 24 bits acpihpet0 at acpi0: 14318179 Hz acpimadt0 at acpi0 addr 0xfee00000: PC-AT compat cpu0 at mainbus0: apid 0 (boot processor) cpu0: Intel(R) Xeon(R) CPU E3-1220 V2 @ 3.10GHz, 3093.38 MHz cpu0: FPU,VME,DE,PSE,TSC,MSR,PAE,MCE,CX8,APIC,SEP,MTRR,PGE,MCA,CMOV, PAT,PSE36,CFLUSH,DS,ACPI,MMX,FXSR,SSE,SSE2,SS,HTT,TM,PBE,SSE3,PCLMUL, DTES64,MWAIT,DSCPL,VMX,SMX,EST,TM2,SSSE3,CX16,xTPR,PDCM,PCID,SSE4.1, SSE4.2,x2APIC,POPCNT,DEADLINE,AES,XSAVE,AVX,F16C,RDRAND,NXE,LONG,LAHF, PERF,ITSC,FSGSBASE,SMEP,ERMS cpu0: 256KB 64b/line 8-way L2 cache cpu0: apic clock running at 99MHz cpu1 at mainbus0: apid 2 (application processor) cpu1: Intel(R) Xeon(R) CPU E3-1220 V2 @ 3.10GHz, 3092.98 MHz cpu1: FPU,VME,DE,PSE,TSC,MSR,PAE,MCE,CX8,APIC,SEP,MTRR,PGE,MCA,CMOV,PAT,PSE36,CFLUSH, DS,ACPI,MMX,FXSR,SSE,SSE2,SS,HTT,TM,PBE,SSE3,PCLMUL,DTES64,MWAIT,DS-CPL, VMX,SMX,EST,TM2,SSSE3,CX16,xTPR,PDCM,PCID,SSE4.1,SSE4.2,x2APIC,POPCNT, DEADLINE,AES,XSAVE,AVX,F16C,RDRAND,NXE,LONG,LAHF,PERF,ITSC,FSGSBASE,SMEP,ERMS cpu1: 256KB 64b/line 8-way L2 cache cpu2 at mainbus0: apid 4 (application processor) cpu2: Intel(R) Xeon(R) CPU E3-1220 V2 @ 3.10GHz, 3092.98 MHz cpu2: FPU,VME,DE,PSE,TSC,MSR,PAE,MCE,CX8,APIC,SEP,MTRR,PGE,MCA,CMOV,PAT,PSE36,CFLUSH, DS,ACPI,MMX,FXSR,SSE,SSE2,SS,HTT,TM,PBE,SSE3,PCLMUL,DTES64,MWAIT,DS-CPL, VMX,SMX,EST,TM2,SSSE3,CX16,xTPR,PDCM,PCID,SSE4.1,SSE4.2,x2APIC,POPCNT, DEADLINE,AES,XSAVE,AVX,F16C,RDRAND,NXE,LONG,LAHF,PERF,ITSC,FSGSBASE,SMEP,ERMS cpu2: 256KB 64b/line 8-way L2 cache cpu3 at mainbus0: apid 6 (application processor) cpu3: Intel(R) Xeon(R) CPU E3-1220 V2 @ 3.10GHz, 3092.98 MHz cpu3: FPU,VME,DE,PSE,TSC,MSR,PAE,MCE,CX8,APIC,SEP,MTRR,PGE,MCA,CMOV,PAT,PSE36,CFLUSH, DS,ACPI,MMX,FXSR,SSE,SSE2,SS,HTT,TM,PBE,SSE3,PCLMUL,DTES64,MWAIT,DS-CPL, VMX,SMX,EST,TM2,SSSE3,CX16,xTPR,PDCM,PCID,SSE4.1,SSE4.2,x2APIC,POPCNT, DEADLINE,AES,XSAVE,AVX,F16C,RDRAND,NXE,LONG,LAHF,PERF,ITSC,FSGSBASE,SMEP,ERMS cpu3: 256KB 64b/line 8-way L2 cache ioapic0 at mainbus0: apid 0 pa 0xfec00000, version 20, 24 pins acpimcfg0 at acpi0 addr 0xe0000000, bus 0-255 acpiprt0 at acpi0: bus 0 (PCI0) acpiprt1 at acpi0: bus 3 (P0P1) acpiprt2 at acpi0: bus 2 (RP01) acpiprt3 at acpi0: bus -1 (RP02) acpiprt4 at acpi0: bus -1 (RP03) acpiprt5 at acpi0: bus -1 (RP04) acpiprt6 at acpi0: bus -1 (RP05) acpiprt7 at acpi0: bus -1 (RP06) acpiprt8 at acpi0: bus -1 (RP07) acpiprt9 at acpi0: bus -1 (RP08) acpiprt10 at acpi0: bus 1 (PEG0) acpiprt11 at acpi0: bus -1 (PEG1) acpiprt12 at acpi0: bus -1 (PEG2) acpiprt13 at acpi0: bus -1 (PEG3) acpicpu0 at acpi0: C3, C2, C1, PSS acpicpu1 at acpi0: C3, C2, C1, PSS acpicpu2 at acpi0: C3, C2, C1, PSS acpicpu3 at acpi0: C3, C2, C1, PSS acpipwrres0 at acpi0: FN00 acpipwrres1 at acpi0: FN01 acpipwrres2 at acpi0: FN02 acpipwrres3 at acpi0: FN03 acpipwrres4 at acpi0: FN04 acpitz0 at acpi0: critical temperature is 106 degC ipmi at mainbus0 not configured cpu0: Enhanced SpeedStep 3093 MHz: speeds: 3101, 3100, 3000, 2900, 2800, 2700, 2600, 2500, 2300, 2200, 2100, 2000, 1900, 1800, 1700, 1600 MHz pci0 at mainbus0 bus 0 pchb0 at pci0 dev 0 function 0 vendor "Intel", unknown product 0x0158 rev 0x09 ppb0 at pci0 dev 1 function 0 "Intel Xeon E3-1200v2 PCIE" rev 0x09: msi pci1 at ppb0 bus 1 mpii0 at pci1 dev 0 function 0 "Symbios Logic SAS2008" rev 0x03: msi mpii0: PERC H200A, firmware 7.15.8.0 IR, MPI 2.0 scsibus0 at mpii0: 42 targets sd0 at scsibus0 targ 1 lun 0: SCSI4 0/direct fixed naa.600508e0000000003cd970ac3a16d00c sd0: 953344MB, 512 bytes/sector, 1952448512 sectors ehci0 at pci0 dev 26 function 0 "Intel 6 Series USB" rev 0x04: apic 0 int 20 usb0 at ehci0: USB revision 2.0 uhub0 at usb0 "Intel EHCI root hub" rev 2.00/1.00 addr 1 ppb1 at pci0 dev 28 function 0 "Intel 6 Series PCIE" rev 0xb4: msi pci2 at ppb1 bus 2 bnx0 at pci2 dev 0 function 0 "Broadcom BCM5716" rev 0x20: apic 0 int 16 bnx1 at pci2 dev 0 function 1 "Broadcom BCM5716" rev 0x20: apic 0 int 17 ehci1 at pci0 dev 29 function 0 "Intel 6 Series USB" rev 0x04: apic 0 int 23 usb1 at ehci1: USB revision 2.0 uhub1 at usb1 "Intel EHCI root hub" rev 2.00/1.00 addr 1 ppb2 at pci0 dev 30 function 0 "Intel 82801BA Hub-to-PCI" rev 0xa4 pci3 at ppb2 bus 3 vga1 at pci3 dev 3 function 0 "Matrox MGA G200eW" rev 0x0a wsdisplay0 at vga1 mux 1: console (80x25, vt100 emulation) wsdisplay0: screen 1-5 added (80x25, vt100 emulation) pcib0 at pci0 dev 31 function 0 "Intel C202 LPC" rev 0x04 ahci0 at pci0 dev 31 function 2 "Intel 6 Series AHCI" rev 0x04: msi, AHCI 1.3 scsibus1 at ahci0: 32 targets ichiic0 at pci0 dev 31 function 3 "Intel 6 Series SMBus" rev 0x04: apic 0 int 19 iic0 at ichiic0 sdtemp0 at iic0 addr 0x19: mcp98243 spdmem0 at iic0 addr 0x51: 8GB DDR3 SDRAM ECC PC3-10600 with thermal sensor isa0 at pcib0 isadma0 at isa0 com0 at isa0 port 0x3f8/8 irq 4: ns16550a, 16 byte fifo com1 at isa0 port 0x2f8/8 irq 3: ns16550a, 16 byte fifo pckbc0 at isa0 port 0x60/5 kbc: cmd word write error pcppi0 at isa0 port 0x61 spkr0 at pcppi0 mtrr: Pentium Pro MTRR support uhub2 at uhub0 port 1 "Intel Rate Matching Hub" rev 2.00/0.00 addr 2 uhidev0 at uhub2 port 1 configuration 1 interface 0 "Avocent USB Composite Device-0" rev 1.10/0.00 addr 3 uhidev0: iclass 3/1 ukbd0 at uhidev0: 8 variable keys, 6 key codes wskbd0 at ukbd0 mux 1 wskbd0: connecting to wsdisplay0 uhidev1 at uhub2 port 1 configuration 1 interface 1 "Avocent USB Composite Device-0" rev 1.10/0.00 addr 3 uhidev1: iclass 3/1 ums0 at uhidev1: 3 buttons, Z dir wsmouse0 at ums0 mux 0 uhub3 at uhub1 port 1 "Intel Rate Matching Hub" rev 2.00/0.00 addr 2 uhub4 at uhub3 port 5 "Standard Microsystems product 0x2514" rev 2.00/0.00 addr 3 vscsi0 at root scsibus2 at vscsi0: 256 targets softraid0 at root scsibus3 at softraid0: 256 targets root on sd0a (f32d1747471a3037.a) swap on sd0b dump on sd0b bnx0: address d4:ae:52:c8:01:89 brgphy0 at bnx0 phy 1: BCM5709 10/100/1000baseT PHY, rev. 8 bnx1: address d4:ae:52:c8:01:8a brgphy1 at bnx1 phy 1: BCM5709 10/100/1000baseT PHY, rev. 8
