---
title: "Mettre en place un serveur de mail avec OpenSMTPD, Dovecot et Rspamd"
date: 2019-12-23 07:57:00 +0200
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
    - Pas de résumé, j'ai passé des heures à traduire, vous allez passer des minutes à lire ;)
    - OK… J'ai expliqué avec BIEN TROP DE DÉTAILS comment mettre en place un serveur de mail
{{< /tldr >}}

Merci à mes sponsors !
--
Un **énorme** merci aux gens qui me sponsorisent sur [patreon](https://www.patreon.com/gilles) ou [github](https://github.com/sponsors/poolpOrg).

Où est-ce que j'ai déjà lu ça ?
--
En Août,
j'ai publié un petit article intitulé "[You should not run your mail server because mail is hard](https://poolp.org/posts/2019-08-30/you-should-not-run-your-mail-server-because-mail-is-hard/)" ("Vous ne devriez pas héberger votre serveur de mail parce que c'est dur") qui était, en gros, mon opinion sur les différentes raisons qui poussent les gens à décourager l'hébergement de mails.
De manière très inattendue, l'article est devenu assez populaire, atteignant 100K lectures et continuant à recevoir des visites et des commentaires plusieurs mois après sa publication.

En Septembre, j'ai publié un autre article beaucoup plus long intitulé "[Setting up a mail server with OpenSMTPD, Dovecot and Rspamd](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/)" ("Installer un serveur de mail avec OpenSMTPD, Dovecot et Rspamd") qui décrivait pas à pas, de façon très détaillée, les étapes pour installer un serveur de mail complet jusqu'à la livraison des e-mails en boîte de réception chez un gros hébergeur de mail américain.
L'article est devenu lui aussi plutôt populaire, sans pour autant atteindre le niveau du précedent article moins technique et spécifique, mais atteignant 40K lectures et continuant également à recevoir des visites et des commentaires plusieurs mois après la publication.
**Le contenu que vous vous apprêtez à lire est la traduction de cet article technique.**

C'est mon second article rédigé en français depuis des années, je vous demande donc un peu d'indulgence :
si vous trouvez des fautes, vous pouvez me les remonter pour que je les corrige, ou faire une [pull request](https://github.com/poolpOrg/poolp.org/) pour les techies.

**Cet article est DENSE** parce que je vais vous tenir la main à un niveau **frisant l'absurdité** en expliquant chacune de mes actions et le pourquoi du comment.
D'un point de vue purement technique, sans explications, l'article pourrait probablement tenir en quelques paragraphes.

N'hésitez pas à **partager la publication** à l'aide des icones en fin d'article et à la commenter <3

Ah, et tant que je vous ai sous la main, je vous recommande chaudement la lecture de mon article non technique ["Décentralisons SMTP pour le bien commun"](https://poolp.org/posts/2019-12-15/decentralisons-smtp-pour-le-bien-commun/) qui aborde certaines des raisons pour lesquelles il nous faut davantage de serveurs de mail.

Bonne lecture !

Cet article est pour les techies et les sysadmins
---
L'auto-hébergement requiert un minimum de connaissances et de motivation.
Ce n'est pas un truc qui se fait en deux clics, il vous faut des connaissances minimales et avoir l'envie d'y dédier un peu de votre temps.
Si vous n'avez jamais touché une zone DNS ou si vous pensez que prendre une heure pour configurer quelque chose est une tâche difficile, alors autant vous prévenir de suite :
**ça ne va pas être simple**.
Je ne vous recommande pas vous lancer dans l'aventure sauf si vous **VOULEZ** avoir à apprendre une tonne de trucs.

Si vous êtes un sysadmin familier avec le travail de sysadmin, alors la plupart de votre travail de sysadmin est déjà considérablement plus dur que mettre en place une instractructure mail.
Pour vous, **le mail ne sera pas quelque chose de difficile**.


EHLO hypno.cat
--
Pour cet article, nous mettrons en place un serveur de mail pour [hypno.cat](https://hypno.cat), un petit site web pour mon (hypothétique) activité prospère d'hypnopraticien.
J'ai enregistré ce domaine il y a plusieurs années parce que j'aimais bien le nom mais il n'a jamais été utilisé pour quoi que ce soit d'autre que l'hébergement d'un **merveilleux** fichier animé.

<center>
  <img src="2019-09-01-newdomain.png">
</center>

Google, Bing et Yahoo le connaissent ; on ne sait pas trop comment *(je vais du moins faire semblant de ne pas savoir)* et ont indexé la page d'accueil, du coup ils ne considèrent pas que ce domaine vient d'être acheté par un spammer pour immédiatement envoyer du mail, mais il reste virtuellement inconnu sur internet parce qu'il n'a aucun contenu, ne fais aucun lien, n'est linké depuis nul part, et n'a jamais émis ou reçu de mail depuis ou vers personne.
C'est un détail plutôt important qui montre que l'âge d'un domaine à un impact sur la réputation, mais ça reste un critère parmi d'autres, on reparlera réputation dans un futur article.

Je vais le mettre en place au fur et à mesure que j'écris cet article, depuis le moment où j'ai démarré un nouveau VPS jusqu'au moment où j'ai pu échanger un mail dans les deux sens avec mon compte Gmail.
J'ai choisi Gmail parce qu'il est sur-représenté dans la population et qu'un nombre important de personnes se sont plains de problèmes de distribution là-bas.
Très personnellement, je pense que Outlook est bien pire en terme d'interopérabilité, mais gardons ça pour un futur article peut-être.

Les services de mail pour [hypno.cat](https://hypno.cat) vont fournir du mail entrant et sortant sécurisés par TLS.
Ils vont permettre de soumettre des messages depuis l'application Gmail de mon smartphone pour ne pas changer les habitudes de mes utilisateurs.
Je pourrais tout aussi bien fournir un webmail, avec [Rainloop](https://www.rainloop.net/) ou [Roundcube](https://www.roundcube.net/), ou installer un client lourd comme [Thunderbird](https://www.thunderbird.net/) ou [mutt](http://mutt.org).
N'importe quel client fonctionnera ; donc je ne vais pas couvrir cette partie. Il y a de nombreuses alternatives et plusieurs articles existants sur internet pour couvrir leur mise en place.
Il y a même des techniques pour assister les clients à automatiquement découvrir leur configuration, mais tout ça reste hors du périmètre de mon article, je ne me fais pas de soucis vous allez trouver tout seul.

Mon mail sortant passera les vérifications basiques de Gmail, plus précisément SPF, DKIM et DMARC, qui sont plus que suffisant pour laisser le mail arriver en inbox.
Il est possible de faire **beaucoup plus**, mais le but de cet article est de trouver le bon équilibre entre faire assez de travail pour avoir bonne allure face aux autres serveurs et rester simple.

Le mail entrant sera filtré pour réduire la quantité de spam, soit en tuant les connexions de mauvais émetteurs évidents dès qu'ils sont détectés, soit en proposant une classification du Spam dans un répertoire dédié pour éviter de saturer la boite de réception.

Je pourrais m'arrêter là parce que c'est déjà un setup assez intéressant, mais je vais rajouter un peu de configuration pour apprendre à Dovecot à entrainer le filtre antispam.
Il apprendra à reconnaitre le Spam lorsque je déplacerai un message dans le répertoire Spam, et à reconnaitre le mail légitime lorsque je déplacerai un message hors du répertoire Spam.
Cette partie est un peu plus complexe parce qu'elle dépend de Sieve qui est loin d'être un bijou d'ingénierie.
Si vous vous moquez de l'apprentissage, vous pouvez sauter cette partie ; j'ai fait sans pendant plus de dix ans. C'est un truc sympa à avoir mais loin d'être une obligation ; c'est juste mon petit cadeau bonus dans cet article.


Pré-requis
--
Le premier pré-requis est de savoir où vous allez faire tourner votre serveur de mail.

Avec la propagation des connexions permanentes type ADSL ou FTTH (fibre), une connexion à domicile peut être tentante mais ce n'est pas une bonne idée parce que les espaces d'adressage IP des FAI sont souvent blacklistés, ou souffrent d'une très mauvaise réputation de départ.
De plus, de nombreux FAI bloquent le trafic SMTP sortant pour éviter que les machines personelles compromises se transforment en bot à spam.
Pour moi, la meilleure option est de louer un serveur dédié ou un VPS depuis une société d'hébergement après s'être assuré que le trafic SMTP sortant est autorisé là-bas.

J'ai loué mes serveurs dédiés chez [online.net](https://www.online.net/) ces douze dernières années et je suis extrêmement satisfait.
Vous trouverez même sur ce blog [des instructions (en anglais) pour faire tourner OpenBSD](https://poolp.org/posts/2018-01-29/install-openbsd-on-dedibox-with-full-disk-encryption/) sur leurs serveurs comme ce n'est pas supporté nativement.
Ils ne filtrent pas SMTP, ce qui est bien parce que vous pouvez faire tourner un serveur de mail immédiatement, mais les adresses IP peuvent avoir été utilisées préalablement par de mauvais émetteurs et il vous faudra les tester.
Si l'adresse IP qui vous est automatiquement allouée n'est pas bonne, soit il va falloir bucher pour gagner en réputation, soit il vous faudra en commander une additionelle en prenant bien le soin de la choisir dans un autre range, après avoir [vérifié qu'elle n'est pas dans une blacklist](https://mxtoolbox.com/blacklists.aspx), ou qu'elle ne [souffre pas déjà d'une mauvaise réputation](https://www.senderscore.org).

Alternativement, pour les besoins d'une offre commerciale que je suis en train de monter, j'ai commencé à construire une infrastructure sur [vultr.com](https://www.vultr.com/?ref=6831037) (lien de parrainage).
Je ne suis là bas que depuis quelques mois et je changerai peut-être d'avis, mais **pour l'instant** j'en suis très satisfait.
Là-bas, SMTP est filtré par défaut et il faut ouvrir un ticket et expliquer quel est le projet pour le trafic mail.
J'ai expliqué que je ne cherchais pas à devenir ESP et que je n'allais pas envoyer de mail commercial de masse mais plutôt gérer de l'hébergement ; ils ont été très serviables et m'ont retiré le filtrage le jour même.

Peu importe où le serveur de mail sera hébergé, ce que vous devez vérifier c'est que : 

- l'hébergeur n'a pas un passif avec l'hébergement de spammers et qu'il autorise SMTP
- vous avez une adresse IP **dédiée au mail**
- vous avez le contrôle du reverse DNS pour cette adresse IP
- votre adresse IP n'est pas déjà sur des blacklists 

Tant que ces pré-requis sont respectés, vous ne devriez pas avoir trop de problèmes.

Comme personne n'aime perdre de mails, il est aussi judicieux de se **préparer aux incidents**.
On s'en protège en utilisant un serveur secondaire pour prendre le trafic lorsque le primaire est down, ou pour re-router le trafic lorsque le primaire est en incapacité de router correctement.
Je ne vais pas couvrir ce sujet parce que ce n'est pas très complexe, il s'agit d'installer un second serveur qui route son trafic vers le premier et de faire le bon enregistrement DNS, après avoir lu cet article vous devriez vous en sortir seul.

**CEPENDANT**, un grand nombre de personnes ont mentionnés s'être déjà fait blacklister et avoir perdu du mail, donc je pense que c'est le bon moment pour donner ce conseil :
**n'hébergez pas vos différents serveurs au même endroit**.
Vous ne voulez pas avoir tous vos serveurs au même endroit s'il y a une panne de courant ou de réseau, et vous voulez aussi éviter que si votre range IP est blacklisté en dommage collatéral d'un voisin malveillant, l'adresse IP de votre serveur secondaire est "**suffisamment éloignée**" pour ne pas être blacklistée aussi.
De cette façon, si votre serveur primaire est en incapacité temporaire d'émettre à cause d'un blocage, vous pouvez re-router votre trafic au-travers de votre serveur secondaire le temps que le problème soit résolu.

J'ai personellement eu à gérer _un seul_ blocage **sur mon propre setup** (bien plus en dehors) durant ces dix dernières années.
C'est un peu chiant parce que les incidents tombent toujours au mauvais moment et que ce n'est jamais fun, mais j'ai re-routé le trafic au-travers d'un serveur secondaire pour le rétablir, rempli un formulaire de délistage en apportant la preuve que j'étais un émetteur légitime, puis re-routé le trafic par mon serveur primaire une fois le problème résolu.
On est **très loin du problème insurmontable** qu'en font un certain nombre de personnes.

**Pas de stress pour ceux qui ont un plan pour les pannes avant qu'elles ne surviennent**.


La stack technique
--
Je vais démarrer un nouveau VPS chez [vultr.com](https://www.vultr.com/?ref=6831037) (toujours un lien de parrainage), installer la dernière version d'[OpenBSD](https://www.openbsd.org)
([snapshot](https://ftp.eu.openbsd.org/pub/OpenBSD/snapshots/amd64/)) parce que c'est mon système de préférence (installation non couverte dans cet article), et construire mon système de mail par dessus.
Je vais partir du principe que nous sommes sous OpenBSD tout le reste de cet article, mais à l'exception de certaines commandes spécifiques pour installer des packages, la configuration est très similaire d'un système à un autre.
Si vous êtes assez grands pour faire tourner un serveur de mail, vous êtes assez grands pour adapter des chemins, remplacer `doas` par `sudo`, et sinon… installez OpenBSD !

<center>
  <img src="https://opensmtpd.org/images/opensmtpd.png">
</center>

Pour la couche SMTP, qui est en charge de transférer des messages entre deux hôtes indépendemment de la manière dont les utilisateurs vont y accéder, j'utiliserais le logiciel [OpenSMTPD](https://www.OpenSMTPD.org/).
Il s'agit du serveur SMTP par défaut pour le système d'exploitation OpenBSD et il y a une version portable [disponible sur Github](https://github.com/OpenSMTPD/OpenSMTPD) avec des instructions pour le construire. Il est également disponible dans [les dépôts de packages de différents systèmes et distributions Linux](https://repology.org/project/opensmtpd/badges), peut-être tout juste à un `pkg`, un `rpm` ou un `apt` de vous.

Les instructions de cet article sont pour la **version 6.6.0** ou ultérieure, elles **ne fonctionneront pas** sur des versions précédentes.
Si votre système ne fournit pas de package à jour, vous pouvez tenter de demander au mainteneur de le mettre à jour ou alors tenter d'en devenir mainteneur.
Il vous faudra construire le package manuellement à partir du dernier tag contenant un `p`, pour `portable`, dans son nom [sur Github](https://github.com/OpenSMTPD/OpenSMTPD/tags).

<center>
  <img src="https://www.dovecot.org/typo3conf/ext/dvc_content/Resources/Public/Images/dovecot_logo_vector.svg">
</center>

Pour la couche IMAP, qui permet aux utilisateurs de récupérer les messages qu'ils ont reçus et y accéder depuis leurs smartphones ou webmail, j'utiliserais la dernière version de [Dovecot](https://www.dovecot.org).
J'utiliserai également le package `Dovecot-Pigeonhole`, qui permettera d'entraîner notre solution antispam en lui faisant apprendre ce qui est du Spam et ce qui est du Ham.
Si vous souhaitez utiliser un client console, en ssh, comme `mutt` par exemple, pas besoin de cette couche car OpenSMTPD peut distribuer dans une boite mail locale à laquelle les clients mail peuvent accéder en direct.
Dans cet article, nous configurerons IMAP parce que nous voulons que les mails puissent être consultés depuis un smartphone comme une personne lambda.

<center>
  <img src="https://camo.githubusercontent.com/ea57696ebeee9a4a6668347acf7ebd8ef8f74ee7/68747470733a2f2f727370616d642e636f6d2f696d672f727370616d645f6c6f676f5f626c61636b2e706e67">
</center>

Enfin, pour la couche de filtrage du spam, j'utiliserai l'excellent [Rspamd](https://www.rspamd.com) qui est bien plus qu'une **simple** solution antispam.
Il fournit des méthodes de filtrage moderne, mais permet également l'intégration de filtrage antivirus, de signature DKIM, et d'une tonne de modules que vous pouvez choisir d'activer ou non pour régler plus finement votre setup.
Dans cet article, nous utiliserons tout simplement ses capacités de filtrage du Spam et de signature DKIM, le strict minimum dont nous avons besoin.


Me rendre accessible
--
SMTP est très étroitement lié avec DNS et les autres hôtes dépendent de recherches DNS pour trouver quelle machine gère le mail de votre domaine.
C'est fait au-travers de la recherche d'enregistrements MX (Mail eXchanger), donc le minimum à faire pour que SMTP fonctionne est de déclarer un enregistrement MX pour votre domaine.
Pour `hypno.cat`, la zone contiendra ce qui suit :

```
;; un enregistrement A (et AAAA pour IPv6) est déclaré pour nommer le serveur de mail

mail.hypno.cat  A       217.69.8.253
mail.hypno.cat  AAAA    2001:19f0:6801:867:5400:01ff:fee7:7af7


;; un enregistrement MX est déclaré pour dire au monde que mail.hypno.cat gère le
;; mail pour hypno.cat
;; 0 est la préférence maximale, si nous avions un serveur secondaire il aurait un
;; nombre supérieur

hypno.cat.      MX 0    mail.hypno.cat.
;;hypno.cat.    MX 10   mail-backup.hypno.cat.
```

Je peux m'assurer que tout est correct en utilisant `host` et `dig` pour vérifier que le nom résoud et que la recherche MX retourne bien le bon nom :

```
$ host mail.hypno.cat
mail.hypno.cat has address 217.69.8.253
mail.hypno.cat has IPv6 address 2001:19f0:6801:867:5400:1ff:fee7:7af7

$ dig -t MX hypno.cat +short
0 mail.hypno.cat.
```

À ce stade et avec ces enregistrements DNS, les autres serveurs de mail peuvent déjà trouver quel serveur est responsable pour `hypno.cat`, et le contacter quand ils veulent envoyer du mail à n'importe quelle adresse mail de ce domaine.


Me faire bien voir
--
Être joignable n'est pas suffisant parce qu'aujourd'hui, vous **DEVEZ** avoir un reverse DNS (rDNS) **ET** un Forward-Confirmed rDNS (FCrDNS).

Il y a une raison derrière cela.
Un grand nombre d'hôtes qui émettent du spam sont des machines compromises, avec une grande portion d'entre elles qui sont des ordinateurs personnels derrière des connexions résidentielles.
Beaucoup de ces connexions n'ont pas de rDNS ou de FCrDNS, ou ont des rDNS qui ressemblent à des patterns d'IP allouées dynamiquement (ex. : `123.123.123.123.dyn.adsl.example.com`).

Comme il s'agit de machines compromises individuellement, et non de machines dont la configuration est sous le contrôle total des spammers, les rDNS et FCrDNS ne peuvent pas être configurés…
il en résulte que la configuration de rDNS et FCrDNS est devenu une preuve de travail : il faut un rDNS et un FCrDNS correctement configuré et ne ressemblant pas à une allocation dynamique.
Chez certains Big Mailer Corps, il s'agit de règles claires, décrites dans les guidelines, pour d'autres on le découvre par la sanction.
Dans tous les cas, il s'agit du **STRICT MINIMUM** ; assurez-vous que vos noms d'hôtes ressemblent à un **VRAI** nom de serveur de mail (ex. : `mail.hypno.cat`, et non pas `www.hypno.cat`).

Le rDNS est généralement hors de votre contrôle parce que dans une zone un peu spéciale, `arpa`, gérée par le propriétaire de l'adresse IP (en général votre hébergeur).
Les FAI ne vous laissent pas toujours les configurer ; mais les hébergeurs de serveurs n'ayant pas vraiment d'autres choix, fournissent généralement un formulaire quelque part sur leurs espaces clients pour assigner un rDNS aux adresses qu'ils vous ont assignées : 

<center>
  <img src="2019-09-01-vultr-rdns.png">
</center>

Si je configure mon rDNS pour avoir la même valeur que l'enregistrement DNS configuré plus haut, alors je passe automatiquement le test du FCrDNS.
On peut le vérifier très simplement en faisant une recherche rDNS pour l'adresse IP :

```
$ host 217.69.8.253   
253.8.69.217.in-addr.arpa domain name pointer mail.hypno.cat.
$ host 2001:19f0:6801:867:5400:1ff:fee7:7af7
7.f.a.7.7.e.e.f.f.f.1.0.0.0.4.5.7.6.8.0.1.0.8.6.0.f.9.1.1.0.0.2.ip6.arpa domain name pointer mail.hypno.cat.
```

Puis cherchez l'adresse IP pour ce rDNS :

```
$ host mail.hypno.cat
mail.hypno.cat has address 217.69.8.253
mail.hypno.cat has IPv6 address 2001:19f0:6801:867:5400:1ff:fee7:7af7
```

Si les valeurs se retrouvent dans les deux sens, tout est d'équerre.


Annoncer quelles machines sont autorisées à émettre pour mon domaine
--
SPF est un mécanisme qui permet pour un domaine destination de déterminer si la machine qui émet était autorisée pour le domaine émetteur.

Le mécanisme est très simple, on ajoute un enregistrement DNS à la zone de notre domaine qui déclare quels serveurs peuvent émettre.
Quand le domaine destination reçoit un mail avec un émetteur de notre domaine, il vérifie si l'adresse IP du serveur émetteur en avait le droit.

SPF n'est pas obligatoire pour SMTP mais c'est une des choses qui nous font paraître légitimes face aux destinataires.
SPF est informatif par défaut ; ne pas passer un test SPF ou ne pas avoir d'enregistrement SPF n'implique pas qu'un mail termine en Spam ou soit détruit.
Il est communément accepté que l'absence d'un enregistrement SPF a un impact très négatif sur la réputation d'un domaine, c'est même explicitement dit dans les guidelines de certains Big Mailer Corps, et vu comme l'enregistrement est simple à fournir… il n'y a vraiment aucune excuse pour ne pas le faire.

Il y a plusieurs façons de configurer SPF ; dans cet example, je vais simplement le configurer pour n'autoriser QUE mon serveur de mail à envoyer du mail en mon nom :
```
hypno.cat.      IN TXT  "v=spf1 mx -all"
```

Si des hôtes reçoivent un mail avec un émetteur `@hypno.cat` qui ne provient pas de `mail.hypno.cat`, ils pourront prendre la décision (ou non) de rejeter le mail.
Ils pourront en tout cas constater qu'il provient d'une machine qui n'était pas autorisée et qu'ils ont le droit de le rejeter.


Prouver que j'ai autorisé le message lui même.
--
DKIM est un mécanisme d'authentification par lequel on peut signer cryptographiquement les mails émis par notre serveur, prouvant qu'on les a vu passer et que l'on a pris la responsabilité de les laisser transiter.
Les hôtes qui recoivent ces mails peuvent vérifier qu'ils étaient bien autorisés en vérifiant la signature, permettant ainsi de détecter des mails forgés hors du système de mail.

Comme SPF, c'est informatif par défaut et l'absence de signature DKIM ou une signature invalide ne veut pas dire qu'un mail ne sera pas distribué. 
En revanche, il est là aussi admis et décrit dans les guidelines que l'absence de DKIM aura un effet sur la réputation d'un domaine.
Et comme configurer DKIM est facile, pas vraiment d'excuse pour ne pas le faire là non plus.

C'est un petit peu plus complexe que pour SPF parce que, même si ça dépend aussi de DNS, il faut générer une paire de clés et publier la clé publique dans l'enregistrement DNS.
Rien de bien foufou non plus.

**Nous sommes en 2019**. Je vais générer une clé RSA de 1024 bits ; **je sais** !

**--- BEGIN DIGRESSION ---**

Les clés RSA 2048 bits ne rentrent pas dans un enregistrement DNS TXT et doivent être tronquées en deux enregistrements, mais tout le monde n'est pas capable de réassocier ces clés tronquées,
et vous pouvez perdre les bénéfices de DKIM si l'hôte destination n'arrive pas à utiliser votre clé publique partielle et considère la signature invalide.

En ce qui me concerne, ces hôtes sont responsables et nous pouvons les ignorer ; mais comme le but de cet article est de permettre une interopérabilité maximale et que DKIM n'est pas présent ici pour sécuriser quoi que ce soit, juste pour vous faire paraître légitime, on va juste regarder ailleurs en justifiant cette mauvaise pratique par le risque modéré de falsification de contenu quand il est utilisé conjointement à SPF.

**Ce n'est pas une bonne manière de raisonner** mais DKIM a d'autres inconvénients et comme virtuellement tout le monde utilise du RSA 1024-bits. Je suggère de laisser le débat 1024 vs >= 2048 à plus tard, lors que nous aurons déjà réussis à échanger correctement avec les Big Mailer Corps.
Je prendrais juste le temps, sans pointer de doigt nul part, d'appuyer sur le fait que certaines sociétés dans la sécurité qui travaillent dans l'industrie du mail ont (avaient ?) encore des problèmes avec les clés 2048-bits en 2019.
Des sociétés qui calculent des scores de réputation.
Oh, la, belle, ironie !

Si c'est un sujet d'inquiétude pour vous, vous pouvez expérimenter la troncature des clé et voir si ça vous convient.
Adapter mon exemple est trivial ; assurez vous juste d'avoir la clé tronquée sur deux enregistrements TXT.

**--- END DIGRESSION ---**


Je crée un répertoire pour contenir les clés :

```
# mkdir /etc/mail/dkim
```

Ensuite, la commande suivante va générer une paire de clé et extraire la clé publique de la clé privée:
```
# openssl genrsa -out /etc/mail/dkim/hypno.cat.key 1024                                       
Generating RSA private key, 1024 bit long modulus
…………………………++++++
……………++++++
e is 65537 (0x10001)

# openssl rsa -in /etc/mail/dkim/hypno.cat.key -pubout -out /etc/mail/dkim/hypno.cat.pub 
writing RSA key

# cat /etc/mail/dkim/hypno.cat.pub                                                                 
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDThHqiM610nwN1nmV8OMc7PaPO
uJWVGRDz5bWj4cRjTTmQYjJQd02xrydRNrRhjEKBm2mMDArNWjoM3jvN04ZifqJx
DmKr7X8jsYi+MPEHz6wZxB8mayDK6glYTCyx//zl1luUdvm26PutA38K8cgnb7iT
kfVP2OqK6sHAdXjnowIDAQAB
-----END PUBLIC KEY-----
```

Enfin, je génére un enregistrement DNS TXT en retirant le blindage de la clé publique et en formattant le contenu pour qu'il soit comme suit :
```
20190913._domainkey.hypno.cat.	IN TXT "v=DKIM1;k=rsa;p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDThHqiM610nwN1nmV8OMc7PaPOuJWVGRDz5bWj4cRjTTmQYjJQd02xrydRNrRhjEKBm2mMDArNWjoM3jvN04ZifqJxDmKr7X8jsYi+MPEHz6wZxB8mayDK6glYTCyx//zl1luUdvm26PutA38K8cgnb7iTkfVP2OqK6sHAdXjnowIDAQAB;"
```

Le nom de l'enregistrement est construit selon le motif : `<selector>._domainkey.<domain>.`, où `<selector>` est un nom arbitraire choisi pour permettre à plusieurs clés de co-exister.
J'aime bien utiliser la date à laquelle j'ai généré la paire de clés, mais peu importe ce que vous choisissez, écrivez le pour plus tard car nous en aurons besoin pour configurer la signature DKIM.

De plus, la clé publique n'a pas besoin de rester dans le répertoire une fois qu'elle est publiée dans l'enregistrement DNS.
Elle peut toujours être dérivée de la clé privée si nous en avons besoin, mais personnellement j'aime bien la garder pour la retrouver rapidement.

**Faites bien attention** à ce que la clé privée ne soit pas en lecture pour tous car, pour une raison inconnue et pas très maline, OpenSSL pense que les clés privées doivent être créées en `rw-r--r--`.


Dire aux autres quoi faire en cas d'erreurs SPF et DKIM avec DMARC
--
SPF et DKIM sont tous les deux très intéressants, mais ils ne sont qu'informatifs, alors ils n'empêchent pas d'abuser de votre nom de domaine.

DMARC permet de dire aux serveurs mails de destination ce que vous voulez qu'ils fassent des mails en apparence de votre domaine qui ratent les tests SPF et DKIM.
Il permet par exemple de leur dire qu'ils doivent rejeter ces mails.

Il n'y a pas d'indicateur clair : fournir une politique de rejet nous fait mieux voir que de fournir une politique sans action. Des expérimentations semblent montrer qu'avoir un enregistrement a un impact positif par rapport à n'en avoir aucun, même si l'enregistrement dit que rien ne doit être fait avec les erreurs SPF et DKIM.
Il serait intéressant de mener une nouvelle expérimentation à ce sujet, mais en même temps, il est plus simple de fournir une politique de rejet et de ne pas avoir à se poser la question :-)

Le format d'un enregistrement DMARC va bien au-delà de cet article et vous trouverez de nombreux exemples dans n'importe quel moteur de cherche. Un enregistrement simple annoncera une politique (p=) de `none` (`reject` si vous voulez rejeter, `quarantine` si vous avez besoin de temps pour décider).
Le champ pourcentage (pct=) déclare combien de ces mails devraient être sujets à la politique DMARC, et le champ Reporting URI of Aggregates (rua=) est la destination des rapports DMARC (pour analyse avant de basculer de `quarantine` à `reject` par exemple).

Je vais ici créer un enregistrement simple, un enregistrement qui fera que les Big Mailer Corps voient que nous nous intéressons à DMARC même si nous ne savons pas encore quoi en faire : 

```
_dmarc.hypno.cat.   IN TXT    "v=DMARC1;p=none;pct=100;rua=mailto:postmaster@hypno.cat;"
```

Mon opinion est qu'il est préférable de basculer rapidement vers `p=reject`, après s'être assuré que les mails sont correctement signés et que seul notre serveur de mail les émet.


Obtenir un certificat pour TLS
--
Il y a plusieurs façons d'obtenir un certificat TLS. 
Je vais partir du principe que vous êtes familiers avec l'hébergement d'autres services et que vous êtes capable d'en obtenir un par votre registrar ou depuis [letsencrypt](https://letsencrypt.org).
Si ce n'est pas le cas, vous trouverez des exemples à foison sur n'importe quel moteur de recherche.

**Si vous savez gérer vos certificats, vous pouvez sauter à la section suivante.**

Comme je suis réellement en train de mettre en place `mail.hypno.cat`, je ne serais pas en mesure de continuer cet article sans obtenir de certificat, donc en bonus pour les utilisateurs d'OpenBSD, je vais documenter la génération de mon certificat sur ma nouvelle installation.

`acme-client` est un utilitaire présent dans le système de base et qui permet de demander ou renouveller des certificats depuis la console.
Il dépend d'un challenge HTTP ; il doit donc être en mesure d'écrire dans un répertoire qui soit accessible par HTTP.
Et comme nous sommes chanceux…
OpenBSD contient un serveur `httpd` que l'on peut utiliser pour ce challenge.
Et comme nous sommmes **TRÈS** chanceux…
OpenBSD fournit aussi un exemple de fichier de configuration que l'on peut utiliser sans gros changement.

On copie simplement le fichier d'example depuis `/etc/example/httpd.conf` vers `/etc/httpd.conf`, en remplaçant `example.com` par `mail.hypno.cat` et en supprimant le bloc TLS puisque seul le challenge nous interesse.
Le fichier doit ressembler à cela : 

```
server "mail.hypno.cat" {
        listen on * port 80
        location "/.well-known/acme-challenge/*" {
                root "/acme"
                request strip 2
        }
        location * {
                block return 302 "https://$HTTP_HOST$REQUEST_URI"
        }
}
```

Le daemon `httpd` peut être démarré comme suit : 
```
# rcctl -f start httpd
httpd(ok)
```

En ce qui concerne `acme-client`, c'est très simple, on copie à nouveau le fichier d'exemple depuis `/etc/example/acme-client.conf` vers `/etc/acme-client.conf`,
en remplaçant `example.com` par `mail.hypno.cat` et on doit se retrouver avec un bloc comme suit : 
```
domain mail.hypno.cat {
        domain key "/etc/ssl/private/mail.hypno.cat.key"
        domain full chain certificate "/etc/ssl/mail.hypno.cat.fullchain.pem"
        sign with letsencrypt
}
```

À ce stade, on se contente de lancer `acme-client` pour notre domaine :
```
# acme-client -v mail.hypno.cat
[…]
acme-client: /etc/ssl/mail.hypno.cat.fullchain.pem: created
```

Le certificat et les clés sont créés au bon endroit, il suffira par la suite d'ajouter les chemins dans OpenSMTPD et Dovecot.

Vous pouvez garder `httpd` en marche pour les renouvellements et appeller `acme-client` depuis un cron, vous pouvez l'éteindre et renouveller d'une autre façon.
Je vous laisse vous débrouiller ; vous trouverez comment faire avec l'aide de la page de manuel `acme-client(1)`.


Installer et configurer Rspamd
--
Sur OpenBSD, Rspamd est packagé et peut être installé d'une simple commande.
Nous avons aussi besoin d'installer Redis qui sert à stocker des statistiques pour Rspamd et son greylisting (entre autres), et également `filter-rspamd` qui est un petit bout de code qui permets à OpenSMTPD de travailler avec Rspamd.
Ils existent également sous forme de packages et peuvent être installés en une seule commande.

Tout d'abord, on installe les packages :
```
mail$ doas pkg_add redis rspamd opensmtpd-filter-rspamd
[…]
redis-4.0.14: ok
rspamd-1.9.0: ok
opensmtpd-filter-rspamd-0.1.1: ok
The following new rcscripts were installed: /etc/rc.d/redis /etc/rc.d/rspamd
See rcctl(8) for details.
```

Rspamd est très configurable, si vous voulez faire des choses un peu funky je vous laisse [lire la documentation](https://rspamd.com/doc/configuration/).
Pour cet article je me contenterai d'une utilisation très basique, comme c'est le cas sur mes propres machines :

Je pourrais éditer la configuration dans `/etc/rspamd/actions.conf` pour ajuster les seuils de mise en spam, mais les valeurs par défaut me conviennent alors je vous les montre juste : 

```
actions {
    reject = 15;    # Reject when reaching this score
    add_header = 6; # Add header when reaching this score
    greylist = 4;   # Apply greylisting when reaching this score (will emit `soft reject action`)
    […]
```

Mais ce que **je veux vraiment** c'est que Rspamd gère la signature DKIM parce que c'est un des pré-requis pour paraître légitime.

On le fait en fournissant une configuration dans `/etc/rspamd/local.d/dkim_signing.conf` pour faire correspondre à notre domaine la clé DKIM générée plus tôt :

```
# mkdir /etc/rspamd/local.d
# cat << EOF >/etc/rspamd/local.d/dkim_signing.conf
allow_username_mismatch = true;

domain {
    hypno.cat {
        path = "/etc/mail/dkim/hypno.cat.key";
        selector = "20190913";
    }
}
EOF
```

La clé de configuration `allow_username_mismatch` est nécessaire parce que Rspamd attend des utilisateurs avec des noms de domaines, mais **dans cette configuration** OpenSMTPD authentifie de simples utilisateurs système.
De plus, faites attention à ce que le fichier contenant la clé DKIM soit autorisé en lecture au groupe `_rspamd`. 

À ce stade on est bon, Rspamd et Redis peuvent être activés pour qu'OpenBSD les démarre à chaque reboot : 

```
# rcctl enable redis
# rcctl enable rspamd
```

Et on peut les démarrer immédiatement pour ne pas avoir à attendre le prochain reboot : 

```
# rcctl start redis
redis(ok)
# rcctl start rspamd
rspamd(ok)
```

Configurer OpenSMTPD
--
OpenSMTPD est installé par défaut sur OpenBSD donc aucune phase d'installation ici.
Si vous utilisez un système d'exploitation différent, soit quelqu'un l'a packagé et vous pouvez l'installer avec votre gestionnaire de paquets, ou vous pouvez le construire depuis les sources et l'installer en utilisant le code depuis le [mirroir Github](https://github.com/OpenSMTPD/OpenSMTPD) et en suivant les instructions de build.

Comme écrit plus haut, **ces instructions ne sont valides que pour la version 6.6.0 et ultérieures**, les versions précédentes ne supportent pas les filtres et certaines des fonctionnalités décrites ici.

La configuration par défault d'OpenSMTPD est adaptée à un serveur de mail local, n'acceptant pas de connexions de l'extérieur, mais capable de laisser les utilisateurs locaux s'échanger des messages et en émettre vers des hôtes distants.

Elle ressemble à ce qui suit :

```
#       $OpenBSD: smtpd.conf,v 1.11 2018/06/04 21:10:58 jmc Exp $

# This is the smtpd server system-wide configuration file.
# See smtpd.conf(5) for more information.

table aliases file:/etc/mail/aliases

# To accept external mail, replace with: listen on all
#
listen on lo0

action "local_mail" mbox alias <aliases>
action "outbound" relay

# Uncomment the following to accept external mail for domain "example.org"
#
# match from any for domain "example.org" action "local_mail"
match from local for local action "local_mail"
match from local for any action "outbound"
```

Je voulais initialement mettre à jour la configuration progressivement pour vous tenir par la main, mais cet article à énormément grossi depuis sa première version.
Par souci de simplicité, je vais simplement mettre les 16 lignes de configuration et les commenter ensuite :

```
pki mail.hypno.cat cert "/etc/ssl/mail.hypno.cat.fullchain.pem"
pki mail.hypno.cat key "/etc/ssl/private/mail.hypno.cat.key"

filter check_dyndns phase connect match rdns regex { '.*\.dyn\..*', '.*\.dsl\..*' } \
    disconnect "550 no residential connections"

filter check_rdns phase connect match !rdns \
    disconnect "550 no rDNS is so 80s"

filter check_fcrdns phase connect match !fcrdns \
    disconnect "550 no FCrDNS is so 80s"

filter senderscore \
    proc-exec "filter-senderscore -blockBelow 10 -junkBelow 70 -slowFactor 5000"

filter rspamd proc-exec "filter-rspamd"

table aliases file:/etc/mail/aliases

listen on all tls pki mail.hypno.cat \
    filter { check_dyndns, check_rdns, check_fcrdns, senderscore, rspamd }

listen on all port submission tls-require pki mail.hypno.cat auth filter rspamd

action "local_mail" maildir junk alias <aliases>
action "outbound" relay helo mail.hypno.cat

match from any for domain "hypno.cat" action "local_mail"
match from local for local action "local_mail"

match from any auth for any action "outbound"
match from local for any action "outbound"
```

C'est tout ce dont nous avons besoin pour notre configuration SMTP complète.

Les deux premières lignes `pki` déclarent que `mail.hypno.cat` utilisera le certificat et la clé que nous avons générés plus tôt pour TLS.

Les lignes `filter` appliquent un ensemble de filtres sur les connexions entrantes. `check_dyndns` filtrera si rDNS corresponds à certains motifs. `check_rdns` filtrera s'il n'y a pas de rDNS.
`check_fcrdns` filtrera si la correspondance FCrDNS échoue.
Il s'agit de filtres "internes" à OpenSMTPD que vous pouvez ajuster pour filtrer différement, à d'autres phases ou avec d'autres critères.

`senderscore` est un filtre custom que vous pouvez installer soit depuis votre gestionnaire de paquets (`pkg_add opensmtpd-filter-senderscore` sur OpenBSD), ou obtenir depuis [Github](https://github.com/poolpOrg/filter-senderscore).
Il n'est pas obligatoire mais je le trouve particulièrement utile et expliquerai pourquoi dans la prochaine section.
Dans cette configuration, il rejettera un émetteur si son senderscore est en dessous de 10, il `junk`-era le message (ajoutera un entête `X-Spam` si le score est inférieur à 70), et pour mon plaisir personnel, ajoutera avant chaque réponse un délai inversement proportionnel à la réputation de l'émetteur pour ralentir les mauvais émetteurs.

`rspamd` est également un filtre custom que nous avons installé dans la précedente section.
Il n'y a aucune configuration car il est entièrement controlé depuis Rspamd, il suffit donc de le déclarer.

Si vous voulez être plus conservateur et ne pas rejeter les mail pour éviter des faux positifs, vous pouvez assigner l'action `junk` plutôt que `disconnect` aux filtres internes et supprimer l'option `-blockBelow` au filtre `senderscore` : 

```
filter check_dyndns phase connect match rdns regex { '.*\.dyn\..*', '.*\.dsl\..*' } junk
filter check_rdns phase connect match !rdns junk
filter check_fcrdns phase connect match !fcrdns junk
filter senderscore proc-exec "filter-senderscore -junkBelow 70 -slowFactor 5000"
```

De cette façon, au lieu de rejeter les sessions, OpenSMTPD va simplement junk les messages pour les faire atterrir en boite à Spam.
En regardant régulièrement ce qui atterri dans cette boite, vous pourrez gagner en confiance et adapter vos réglages pour trouver ceux qui vous conviennent.

Ces filtres simples, sans même compter `rspamd`, sont suffisants pour tuer la plus grosse partie du spam.
Chez moi, ils permettent de passer de _plusieurs centaines_ de spam quotidiens à une poignée.
Le filtre `rdns regex` doit être adapté en fonction du pays, vous pouvez trouver des motifs qui sont récurrents chez vous et n'apparaissent jamais chez moi.
Pour faciliter la maintenance, ils peuvent être stockés dans un fichier à raison d'un par ligne, en respectant la construction suivante : 

```
table <dyndns> file:/etc/mail/path-to-my-regex-file
filter check_dyndns phase connect match rdns regex <dyndns> junk
```

Continuons à disséquer la configuration.

La table `aliases` pointe vers un fichier d'alias sur deux colonnes.
La colonne de gauche correspond à la partie utilisateur d'une adresse e-mail que vous recevez ; la colonne de droite correspond à l'utilisateur à qui le mail doit être livré (ex. : `root: gilles`).
Il est recommandé d'aliaser les adresses `root`, `abuse` et `postmaster` vers une adresse que vous lisez réellement.
Je ne le fais pas ici parce que je vais simplement détruire le setup une fois l'article fini, mais en ce qui vous concerne, faites le !

Ensuite viennent les lignes `listen`.
La première est le point d'attache que vont contacter les autres serveurs de mail qui souhaitent envoyer du mail vers `hypno.cat`. Elle est configurée pour offrir `tls` en utilisant les certificats et clés pour `mail.hypno.cat` et pour filtrer les sessions entrantes avec mes filtres.
La seconde est le point d'attache que vont contacter mes propres utilisateurs sur le port de `submission`, qui requiert TLS en utilisant les mêmes certificats et clés, mais également l'authentification des utilisateurs système locaux (le défaut peut être changé), et filtrant à-travers le filtre rspamd uniquement dans le but de faire la signature DKIM de nos messages.

Les lignes `action` définissent les actions à entreprendre avec les mails qui sont acceptés dans le système.
Une action locale permet de distribuer dans une maildir, en classifiant les spams dans un répertoire spécifique, et en résolvant les alias.
Une seconde action permets de relayer les mails en s'annonçant en tant que `mail.hypno.cat` aux autres hôtes.
Si le serveur disposait de plusieurs adresses IP, il serait possible d'assigner à l'aide de l'option `src` celle à utiliser, mais ici je n'ai qu'une seule adresse IP donc pas besoin de la spécifier.

Enfin, les lignes `match` représentent l'ensemble des règles.
Quand une enveloppe entre dans le serveur SMTP au travers d'un des points d'attache `listen`, elle est comparée séquentiellement à chaque ligne `match` jusqu'à trouver une correspondance ou rejetée si aucune correspondance n'est trouvée.
Lorsqu'une correspondance est trouvée, l'action associée à la règle est prise en compte.
Les règles de match sont très simples à comprendre, je ne vais donc pas les expliquer ; vous devriez arriver à comprendre seuls ce que signifie `from any for domain "hypno.cat"`, sans quoi vous ne devriez pas lire cet article en premier lieu.


Quelques mots au sujet de SenderScore
----
SenderScore est une base de donnée de réputation IP qui associe un score entre 0 et 100 à une adresse IP.
Le score est lié au volume et comportement observé depuis ces adresses IP, et même si l'on ne sait pas COMMENT sont calculés ces scores parce que la méthodologie n'est pas publique,
un peu d'étude permet de confirmer que ces nombres ne sortent pas de nul part : ils sont corrélés au ratio de distribution et d'erreur chez différents Big Mailer Corps et il est possible d'influencer le score en changeant de comportement.

Il est également **TRÈS** clair que pour construire les scores de réputation, ils ont accès à des informations relatives aux sessions SMTP dont seuls les serveurs mails de destinations peuvent disposer, comme le volume de mails ou le ratio de destinataires acceptés et refusés, observés depuis une adresse, impliquant que ces informations sont obtenues directement depuis les Big Mailer Corps.
Faites ce que vous voulez de cela.

En me basant sur **_MA_** compréhension, il s'agit d'un indicateur intéressant pour limiter la quantité de spam qui vous atteint.
Je ne fait pas confiance à cet indicateur concernant les bons scores (parce que je sais à peu près contourner) mais j'y fais ÉNORMÉMENT confiance en ce qui concerne les mauvais scores, parce qu'il faut vraiment faire un paquet de mauvaises choses en gros volume pour que le score se dégrade.
Si vous ne faites pas de mailing en masse, vous êtes censés être soit inconnu de cette base de donnée ou avoir un score supérieur à 95, sinon il y a quelque chose qui déconne chez vous.

Tout le monde n'est pas convaincu par SenderScore et certains experts en déliverabilité disent que c'est du pipeau.
J'ai personnellement fait tourner une expérimentation sur plusieurs mois en graphant volume et réputation quotidienne, en comparant aux graphs de distribution. Mon opinion est contraire : 
il y a une correlation très significative qui permet clairement de classifier les émetteurs.
Je pense que la meilleure approche est d'utiliser le filtre pour junker, et non bloquer, pour déterminer par vous même si vous en êtes satisfait.

SenderScore considère que les émetteurs devraient avoir une réputation supérieure à 70. Je pense pour ma part que les hôtes avec un score inférieur à 70 sont de bons candidats au junk, les hôtes avec un score inférieur à 10 sont des rejets évidents.


Installer et configurer Dovecot
--
OpenBSD dispose d'un paquet pour Dovecot qui peut être installé d'une simple commande : 

```
mail# pkg_add dovecot
dovecot-2.3.7.2v0: ok
The following new rcscripts were installed: /etc/rc.d/dovecot
See rcctl(8) for details.
New and changed readme(s):
        /usr/local/share/doc/pkg-readmes/dovecot
[…]
```

Et, **specifiquement pour OpenBSD**, en suivant les instructions du fichier readme, le fichier `/etc/login.conf` devrait contenir une classe `dovecot` pour augmenter les ressources autorisées à Dovecot, gourmand en descripteurs de fichiers : 

```
dovecot:\
    :openfiles-cur=1024:\
    :openfiles-max=2048:\
    :tc=daemon:
```

Cela fait, il n'y a plus grand chose à faire à part pointer Dovecot sur le certificat et la clé TLS générés plus tôt. 
On le fait en modifiant les clés de configuration `ssl_cert` et `ssl_key` dans le fichier `/etc/dovecot/conf.d/10-ssl.conf` pour qu'elles soient comme suit :

```
ssl_cert = </etc/ssl/mail.hypno.cat.fullchain.pem
ssl_key = </etc/ssl/private/mail.hypno.cat.key
```

Puis dire à Dovecot que les mails sont livrés dans le répertoire `~/Maildir` de chaque utilisateur puisque c'est là que OpenSMTPD les déposera.
On le fait en modifiant la clé de configuration `mail_location` dans le fichier `/etc/dovecot/conf.d/10-mail.conf` pour qu'elle soit comme suit : 

```
mail_location = maildir:~/Maildir
```

On est bon. 

Le daemon peut être activé pour être démarré à chaque reboot, puis démarré immédiatement :

```
mail# rcctl enable dovecot
mail# rcctl start dovecot   
dovecot(ok)
```

À ce stade, il est possible de configurer n'importe quel client mail comme `mutt`, `thunderbird` ou même l'application `gmail` sur Android, et utiliser `mail.hypno.cat` pour les mails entrants et sortants.

<center>
  <img src="2019-09-01-smartphone.png">
</center>


Apprendre à Dovecot à entrainer Rspamd
---
Vous avez lu jusqu'ici ? bien.

Cette section est une section bonus, elle n'est absolument pas nécessaire pour configurer votre serveur de mail, mais c'est un élément que j'aime ajouter parce qu'il permet aux utilisateurs d'entrainer le filtre antispam d'une manière à laquelle ils sont habitués : 
en déplaçant un mail vers la boite à Spam, ou en appuyant sur un bouton "signaler un Spam", "marquer comme Spam" ou peu importe comment il s'appelle dans l'interface utilisée. Le bon côté est que cet élément de configuration s'intègre avec IMAP, donc peu importe comment les mails sont consultés et depuis quel logiciel, l'action de marquer un mail comme Spam entraînera automatiquement le filtre.

C'est très simple en théorie : il suffit de brancher deux scripts, l'un qui gère le déplacement dans la boite à Spam, l'autre qui gère le déplacement hors de la boite à Spam.
Les Big Mailer Corps utilisent des stratégies plus poussées en détectant les mails non ouverts, ceux déplacés directement en poubelle, etc… 
mais débutons simple, il est toujours possible d'étendre ensuite.

Pour faire ce branchement, nous dépendont de Sieve qui est… 
comment dire avec diplomatie… 
un peu sur-conçu.

**Ce n'est pas un élément critique du setup, vous êtes autorisés à éteindre vos cerveaux et suivre les instructions aveuglément.**
**Le pire qui puisse arriver ici est que l'entraînement de la détection ne marche pas.**

Pour activer imap sieve, le package Pigeonhole doit être installé, encore une fois à l'aide d'une simple commande :

```
mail# pkg_add dovecot-pigeonhole
dovecot-pigeonhole-0.5.7.2v1: ok

[…]
```

Et… voici la partie la plus complexe, il faut configurer Dovecot pour activer `imap_sieve` et lui apprendre quels scripts utiliser pour quelles actions.

D'abord, on active `imap_sieve` dans Dovecot en l'ajoutant dans la liste des plugins mails pour IMAP.
C'est fait en mofifiant la clé de configuration `mail_plugins` dans le fichier `/etc/dovecot/conf.d/20-imap.conf` : 

```
protocol imap {
    […]

    mail_plugins = $mail_plugins imap_sieve
 
    […]
}
```

Ensuite pour apprendre à Dovecot à entrainer Rspamd, on ajoute la section suivante dans le fichier `/etc/dovecot/conf.d/90-plugin.conf`, pour dire quel script sieve déclencher quand un mail est déplacé vers ou hors du répertoire Junk : 

```
plugin {
  sieve_plugins = sieve_imapsieve sieve_extprograms
  sieve_global_extensions = +vnd.dovecot.pipe +vnd.dovecot.environment

  imapsieve_mailbox1_name = Junk
  imapsieve_mailbox1_causes = COPY APPEND
  imapsieve_mailbox1_before = file:/usr/local/lib/dovecot/sieve/report-spam.sieve

  imapsieve_mailbox2_name = *
  imapsieve_mailbox2_from = Junk
  imapsieve_mailbox2_causes = COPY
  imapsieve_mailbox2_before = file:/usr/local/lib/dovecot/sieve/report-ham.sieve

  imapsieve_mailbox3_name = Inbox
  imapsieve_mailbox3_causes = APPEND
  imapsieve_mailbox3_before = file:/usr/local/lib/dovecot/sieve/report-ham.sieve

  sieve_pipe_bin_dir = /usr/local/lib/dovecot/sieve
}
```

Maintenant que Dovecot est prêt, on prépare la partie Sieve qui dépend de deux scripts pour entraîner le Spam et le Ham dans `/usr/local/lib/dovecot/sieve` : 

```
# cat report-ham.sieve

require ["vnd.dovecot.pipe", "copy", "imapsieve", "environment", "variables"];

if environment :matches "imap.mailbox" "*" {
  set "mailbox" "${1}";
}

if string "${mailbox}" "Trash" {
  stop;
}

if environment :matches "imap.user" "*" {
  set "username" "${1}";
}

pipe :copy "sa-learn-ham.sh" [ "${username}" ];
```

```
# cat report-spam.sieve

require ["vnd.dovecot.pipe", "copy", "imapsieve", "environment", "variables"];

if environment :matches "imap.user" "*" {
  set "username" "${1}";
}

pipe :copy "sa-learn-spam.sh" [ "${username}" ];
```

Et parce que les deux scripts Sieve dépendent de scripts shell `sa-learn-ham.sh` et `sa-learn-spam.sh`, on crée aussi ces deux scripts shell dans `/usr/local/lib/dovecot/sieve` : 

```
# cat sa-learn-ham.sh

#!/bin/sh
exec /usr/local/bin/rspamc -d "${1}" learn_ham
```

```
# cat sa-learn-spam.sh

#!/bin/sh
exec /usr/local/bin/rspamc -d "${1}" learn_spam
```

Enfin, on compile les scripts Sieve et on rend les scripts shell exécutables pour que Dovecot puisse les utiliser : 

```
# sievec report-ham.sieve
# sievec report-spam.sieve
# chmod 755 sa-learn-ham.sh
# chmod 755 sa-learn-spam.sh
```

C'est tout. Dorénavant en déplaçant un mail d'un répertoire à un autre, on doit voir les lignes suivantes dans le fichier de log `/var/log/rspamd/rspamd.log` : 

```
2019-09-13 23:59:46 #18598(controller) <1d44bd>; csession; rspamd_controller_learn_fin_task: <127.0.0.1> learned message as ham: CAHPtQbOxQxBCsVd7nUCP4podu74Pa-F6k28z+4BWfNeeqWYiAg@mail.gmail.com
[…]
2019-09-14 00:01:57 #18598(controller) <b76e28>; csession; rspamd_controller_learn_fin_task: <127.0.0.1> learned message as spam: CAHPtQbOxQxBCsVd7nUCP4podu74Pa-F6k28z+4BWfNeeqWYiAg@mail.gmail.com
[…]
```

Il est surement possible de simplifier, je vais être honnête et dire que mon intérêt pour Sieve n'est pas assez poussé pour que je me perfectionne.
C'est une fonctionnalité sur le côté et je ne toucherai probablement pas les scripts avant les dix prochaines années.
Je ne suis pas du tout convaincu par Sieve et quand j'aurai le temps, je ferais une alternative qui ne me rappelle pas les années m4 de Sendmail.


Tester le tout
--
Et maintenant, on va juste tester que l'on arrive à faire un trajet complet aller-retour de mail depuis `hypno.cat` vers `gmail.com`.

Tout d'abord, je prépare un mail depuis mon compte `hypno.cat` vers mon compte `gmail.com` : 

<center>
  <img src="2019-09-01-craft-mail.png">
</center>

Après envoi, je vérifie qu'il arrive bien chez `gmail.com` : 

<center>
  <img src="2019-09-01-gmail-1.png">
</center>

Ensuite, je vérifie que l'émission a bien eu lieu par dessus un canal sécurisé par TLS : 

<center>
  <img src="2019-09-01-gmail-2.png">
</center>

Je vérifie également que `gmail.com` est content avec notre déclaration SPF, notre signature DKIM et qu'il a bien vu notre enregistrement DMARC.
On fait cela en sélectionnant "Voir l'original" dans le menu associé à chaque mail : 

<center>
  <img src="2019-09-01-gmail-3.png">
</center>

Enfin, j'y répond pour être sur que ça fonctionne dans les deux sens : 

<center>
  <img src="2019-09-01-received-mail.png">
</center>


Cool, on a fini
--
Cet article est dense, je voulais _expliquer_ pourquoi on fait les choses. Si vous retirez toutes les explications et gardez juste les détails purement techniques, vous réaliserez que nous avons construit une infrastructure de mail, fournissant des services entrants et sortants sécurisés par TLS, avec un trafic sortant respectant DKIM, SPF et DMARC, et un trafic entrant filtrant le spam.

On a fait cela en :

- ajoutant deux enregistrements A, un MX et trois TXT dans notre zone DNS;
- faisant attention à avoir un rDNS correctement configuré
- installant un certificat TLS
- préparant un fichier de configuration d'environ 15 lignes pour un serveur SMTP
- modifiant 3 lignes dans la configuration par défaut d'un serveur IMAP
- modifiant 8 lignes dans la configuration par défaut d'une solution antispam

Et parce que nous étions de tres bonne humeur et volontaires pour faire un petit pas de plus, nous avons implémenté un apprentissage Spam / Ham de Rspamd en : 

- modifiant environ 20 lignes dans la configuration par défaut d'un serveur IMAP
- copiant 4 scripts Sieve dans un répertoire

On admettra qu'un nouvel arrivant va devoir fournir quelques efforts pour trouver tout ça sans aide, mais aucune de ces tâches n'est réellement difficile ou complexe.
De plus, la plupart sont des opérations de mises en place qui ne sont réalisées qu'une fois ; on n'édite pas la zone DNS ou un reverse DNS tous les deux jours, tout comme on ne touche pas une configuration en place qui fonctionne tant qu'il n'y a pas de nouveau cas d'utilisation.

Pour citer les plus réfractaires à l'auto-hébergement, il peut y avoir des maintenances pour gérer les conséquences d'une blacklist, mais il n'y a aucun scénario dans lequel ladite maintenance impliquerait autant de travail que la mise en place que nous venons de faire… et qui au final se réplique en 10 minutes chrono une fois réalisée une première fois pour se faire la main.


Quelle est la prochaine étape ?
--
Votre prochaine étape est de mettre en place de la redondance pour vous assurer qu'une panne de votre serveur de mail n'entraine pas de perte de mail.

En pratique, la plupart des serveurs de mails retentent une émission de multiples fois si la destination est inaccessible donc, **même** en cas de panne, la plupart de vos mails seront temporisés par votre émetteur et retransmis quand votre serveur sera de nouveau accessible.

En théorie, on veut tout de même faire les choses bien, et reposer sur la responsabilité des autres à gérer nos pannes n'est pas très poli.
Vous devez mettre en place un serveur secondaire, vous arranger entre amis pour être serveurs secondaires les uns des autres, ou même vous souscrire à un service qui offre du mail secondaire.
N'importe quoi qui puisse permettre de couvrir vos arrières lorsque votre serveur primaire tombe.

Ensuite, peu importe la méthode choisie, il vous faudra : 

- ajouter un enregistrement MX supplémentaire dans votre zone DNS
- configurer le serveur secondaire pour retransmettre les mails au serveur primaire

On est loin de la haute voltige.


Est-ce qu'on a vraiment fini ?
--
Je vais probablement écrire une série d'article pour discuter de la réputation, ainsi que des mécanismes en place chez les Big Mailer Corps qui provoquent des pannes de distribution.

Si ce sont des sujets d'intérêt et que j'ai du feedback positif, j'écrirais plus souvent sur les concepts autour de la déliverabilité, sinon je reprendrais mes anciennes activités : écrire mes rapports mensuels sur mes contributions opensource. À vous de me dire :-)

Finalement, dans cet article j'ai décrit un setup vraiment simple mais il y a des tonnes de choses intéressantes à faire avec les infrastructures de mail, si vous n'avez pas peur d'aller plus loin.

Je construis actuellement une infrastructure de mail pour un future service commercial d'hébergement.
L'infrastructure s'étend sur plusieurs data centers dans plusieurs pays, a des serveurs secondaires capable d'encaisser des pannes très sérieuses des serveurs primaires, peut facilement monter à l'échelle sur des pics de volume, elle décorelle le trafic entrant et sortant pour permettre de fournir des services partiels, peut facilement re-router le trafic entre des machines pour contourner des pannes, tout en fournissant des services IMAP à des comptes virtuels sur une multitude de domaines, avec des backups quotidiens.
L'infrastructure me coute… moins de 75 EUR par mois.
Elle est également très simple, la plupart des machines sont des installations par défaut d'OpenBSD avec **très peu** de changements au système de base.

Si lire à propos de ce genre de mises en place vous intéresse, je peux écrire à ce sujet pour aider un maximum de personnes à monter un maximum d'alternatives aux Big Mailer Corps, ce qui est au final mon souhait.

Merci de m'avoir lu, je n'ai pas de soundcloud, mais [j'ai un patreon](https://patreon.com/gilles) et un [github](https://github.com/sponsors/poolpOrg) si vous voulez me sponsoriser, ou **utiliser les icones en dessous du lien de commentaires** pour partager cet article sur les reseaux sociaux.

