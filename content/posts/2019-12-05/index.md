---
title: "Décentralisons SMTP pour le bien commun"
slug: decentralisons-smtp-pour-le-bien-commun
date: 2019-12-15 13:05:00 +0200
author: Gilles Chehade
categories:
 - technology
---

{{< tldr >}}
    - SMTP est la méthode dont les ordinateurs échangent des e-mails
    - il s'agit d'un protocole décentralisé, ce qui signifie que CHACUN peut héberger un nœud et être indépendant
    - il est en train d'être centralisé dans des sociétés qui ont un passif d'abus
    - il est en train d'être centralisé dans un pays qui a un passif d'abus
{{< /tldr >}}

Où est-ce que j'ai déjà lu ça ?
--
En Août,
j'ai publié un petit article intitulé "[You should not run your mail server because mail is hard](https://poolp.org/posts/2019-08-30/you-should-not-run-your-mail-server-because-mail-is-hard/)" ("Vous ne devriez pas héberger votre serveur de mail parce que c'est dur") qui était, en gros, mon opinion sur les différentes raisons qui poussent les gens à décourager l'hébergement de mails.
De manière très inattendue,
l'article est devenu assez populaire,
atteignant 100K lectures et continuant à recevoir des visites et des commentaires plusieurs mois après publication.

En Septembre,
j'ai publié un autre article beaucoup plus long intitulé "[Setting up a mail server with OpenSMTPD, Dovecot and Rspamd](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/)" ("Installer un serveur de mail avec OpenSMTPD, Dovecot et Rspamd") qui décrivait pas à pas,
de façon très détaillée,
les étapes pour installer un serveur de mail complet jusqu'à la livraison des e-mails en boîte de réception chez un gros hébergeur de mail américain.
L'article est devenu lui aussi plutôt populaire,
sans pour autant atteindre le niveau du précedent article moins technique et spécifique,
mais atteignant 40K lectures et continuant également à recevoir des visites et des commentaires plusieurs mois après publication.

Le contenu que vous vous apprêtez à lire est extrait de ce second article auquel il n'aurait pas dû être rattaché.
Il est bien trop (géo-)politique pour faire partie d'un article technique,
j'ai donc décidé de l'en retirer et d'en faire un dédié.
Je ne veux pas que les spécificités d'un article sur **OpenSMTPD** aille à l'encontre d'un message beaucoup plus général.

C'est mon premier article en français depuis des années,
je vous demande donc un peu d'indulgence : 
si vous trouvez des fautes,
vous pouvez me les remonter pour que je les corrige,
ou faire une pull request pour les techies.

N'hésitez pas à **partager la publication** à l'aide des icones en fin d'article et à la commenter <3


Qui sont Big Mailer Corps ?
--
Je ne vais pas pointer de doigts vers des directions en particulier.
Concrètement,
rentrent dans ma définition des Big Mailer Corps,
toute grosse entreprise qui centralise l'hébergement des e-mails d'un très grand nombre d'utilisateurs : 

    - Orange ? SFR ? La Poste ?
    - Non, plus grand, plus mondial, …

Et là, normalement le doute est dissipé,
pas besoin de citer de nom.
Vous avez compris qui est visé parce que **TOUT LE MONDE** connait les Big Mailer Corps,
la plupart d'entre vous avez une adresse chez l'un d'entre eux,
et la plupart de vos contacts sont hébergés chez l'un d'entre eux.


L'auto-hébergement et encourager les petits fournisseurs est meilleur pour notre bien commun
--
Il y a des **conséquences** à la centralisation des services de mail chez les Big Mailer Corps.

Il n'y a aucun sens à ce que Jacques Lambda,
qui partage des photos de chatons avec sa famille et ses amis,
se construise une infrastructure d'hébergement de mails alors que plusieurs Big Mailer Corps lui offrent,
"gratuitement",
une qualité de service au top.
Il peut obtenir une adresse e-mail immédiatement disponible et qui marche de manière parfaitement fiable.
Il n'y a absolument aucun sens à ce que Jacques Lambda n'aille pas chez l'un des Big Mailer Corps,
particulièrement quand **même les techies** font ce choix sans hésitation,
confirmant que c'est plutôt une bonne option.

Il n'y a rien de mal à ce que tous les Jacques Lambda choisissent un service qui marche.

Ce qui est **terriblement mauvais** par contre,
c'est le centralisation d'un protocole de **communication** dans les mains d'une poignée d'entreprises commerciales,
**CHACUNE D'ENTRE ELLES** provenant du même pays (actuellement dirigé par un lunatique qui abuse de son pouvoir et [souffre probablement de trouble de la personnalité narcissique](https://psychcentral.com/blog/the-psychology-of-donald-trump-how-he-speaks/)),
**CHACUNE D'ENTRE ELLES** étant apparu dans les informations et/ou dans une cour de tribunal pour tout un assortiment de comportements "déplaisants" (abus de la vie privée, espionnage, abus de monopole, harcellement professionnel ou sexuel, j'en passe… ),
et **CHACUNE D'ENTRE ELLES** faisant croître des bases d'utilisateurs qui **dépassent de loin la population totale de plusieurs pays combinés**.

Mettons un peu de perspective.
Le plus gros des Big Mailer Corps annonce une base d'utilisateur qui dépasse **1.4 MILLARD** d'utilisateurs (Avril 2018),
soit à peu près **la population totale** de la Chine ou de l'Inde,
plus de **quatre fois** la population totale des USA,
ou plus de **vingt fois** la population totale de la **France**.
Si vous comptiez les utilisateurs au rythme d'un par seconde,
**il vous faudrait 44 ans pour en faire le tour**.

Ensuite,
**très loin derrière**,
il est suivi par le prochain Big Mailer Corp qui annonce une base de **400 millions** d'utilisateurs (2018 également),
dépassant la population totale des USA,
atteignant **quatre fois** la population totale de l'Égypte,
et **cinq fois** la population totale de la **France**.

Dans la plupart des rapports de mailing que j'ai pu surveiller,
le premier des Big Mailer Corps couvre approximativement la moitié des adresse e-mails de destinataires…
l'autre moitié étant également couverte en large partie par __les autres__ Big Mailer Corps.

**Ce n'est pas sain**.
Ni ici,
ni nulle part,
ni maintenant,
ni jamais,
ni dans aucune dimension alternative _(insérer "sliiiiiiders" en chuchotement mental ici)_.

<center>
<img src="feature.jpg">
</center>

Prenez un moment pour bien enregistrer ce qui suit : 

Si ces grosses entreprises étaient contraintes de couper les communications avec un pays pour appliquer des sanctions,
bien au delà d'un **MILLIARD** de personnes seraient hors de portée pour le pays visé.
Et si vous pensez la crainte exagérée,
les utilisateurs de Github en Iran,
en Syrie,
à Cuba ou en Crimée pourraient vous donner un point de vue différent après
[avoir découvert qu'ils étaient mis à la porte](https://techcrunch.com/2019/07/29/github-ban-sanctioned-countries/) pour appliquer les sanctions américaines sur leurs pays.

**Aussi ennuyant soit-il**,
Git reste un outil de techies qui n'impacte principalement que des techies,
une minorité d'humains qui sait trouver facilement des solutions de contournement…
facilitées par le fait que Git est décentralisé et ne nécessite pas d'interconnexions et d'interopérabilité : 
les utilisateurs peuvent facilement bouger ailleurs et s'auto-héberger.
Aucun lien coupé avec aucun pays ne peut empêcher ça.

Le mail fonctionne de façon différente.
Il impacte **tout le monde**,
des personnes de tous âges et qui ne sont pas forcément des techies.
Il est toujours possible de trouver une solution de contournement en cas de blocage,
mais ce n'est pas simple pour tout le monde.
Et comme il s'agit d'un protocole pour mettre en relation un émetteur A avec un destinataire B,
il y a un besoin d'interconnexion, un besoin d'interopérabilité ET besoin que le lien entre A et B ne soit pas coupé.
Les conséquences d'un blocage similaire à celui de Github chez les Big Mailer Corps seraient plusieurs ordres de magnitude plus durs et visibles : **un pan des utilisateurs d'Internet ne serait tout bonnement plus joignable par les citoyens des pays sanctionnés**.

Les habitants des [pays de merde](https://www.washingtonpost.com/politics/trump-attacks-protections-for-immigrants-from-shithole-countries-in-oval-office-meeting/2018/01/11/bfc0725c-f711-11e7-91af-31ac729add94_story.html) ne sont pas tous des techies,
ils sont des personnes qui ont **BESOIN** d'interconnexions et d'interopérabilité pour communiquer avec plus d'un
**MILLIARD** de personnes hébergées chez les Big Mailer Corps.
Le mail permet de connecter les populations mondiales de manière fiable…
**UNIQUEMENT** si le mail reste **décentralisé**.
Il ne marche plus s'il est centralisé dans une poignée d'entreprises toutes localisées dans le même pays.

Bien sûr cela n'arrivera peut être jamais,
et je metterai même mon billet dessus parce qu'
[un pays connu pour avoir intercepté et épié les communications mondiales](https://en.wikipedia.org/wiki/PRISM_(surveillance_program))
a plus d'intérêt à conserver le flot des communications qu'à le couper,
mais le seul fait que ce soit **_TECHNIQUEMENT_** possible devrait être suffisant pour nous mettre mal à l'aise.
J'ignore même volontairement que la raison pour laquelle ils ne le feront pas me rends également mal à l'aise.
Si les Big Mailer Corps étaient hébergés en Chine,
nous serions déjà en train d'envisager les conséquences d'une coupure et d'envisager un plan de sortie,
mais nous ne le faisons pas parce que nous _pensons_ que nous sommes du bon côté de la barrière,
et que [des sanctions à notre encontre ne pourraient pas arriver](https://www.huffingtonpost.fr/entry/contre-les-sanctions-de-trump-la-france-et-leurope-riposteront_fr_5d95aac7e4b0f5bf796f7feb).

Mais ça,
c'est jusqu'à ce que l'on soit complètement au pied du mur sans solution.

À ce stade,
je pourrais également faire **semblant d'ignorer** que le modèle économique de la plupart de certaines de ces entreprises repose sur la création d'un profil commercial de leurs utilisateurs et qu'avoir à disposition tous les e-mails est une mine d'or.

Certains Big Mailer Corps se **[défendent](https://www.lesechos.fr/2017/06/google-nexploitera-plus-gmail-pour-faire-de-la-pub-ciblee-174374)** d'exploiter le contenu des e-mails,
mais…
il n'y a **pas besoin de connaître le contenu des e-mails** si les expéditeurs et destinataires sont **écrits sur l'enveloppe**.
Il est tout à fait possible de savoir que vous essayer de souscrire un crédit en faisant des comparatifs,
mais que vous recevez également des remboursements de santé réguliers et que vous recevez des informations concernant un certain type d'appareillage médical. Un profil très précis et qui serait très intéressant pour des organismes de prêt ou des assurances peut tout à fait être dressé sans avoir à "lire" les e-mails…
mais bon, ce ne sont que des spéculations basées sur **quelque chose de techniquement réalisable**.

Dans la même idée,
je vais faire **semblant d'ignorer** que tous les documents partagés peuvent être analysés pour détecter des logiciels malveillants,
que le document soit confidentiel ou non,
qu'il soit stratégique ou non.
Tout comme les liens échangés par mail,
qu'ils soient stratégiques ou non,
perdent tout caractère confidentiel lorsqu'ils [passent par le webmail d'un Big Mailer Corp](https://www.npr.org/sections/thetwo-way/2013/08/14/212034992/gmail-users-shouldnt-expect-privacy-google-says-in-filing?t=1576417433323).


Alors **OUI**,
il faut plus de personnes qui s'auto-hébergent ou **qui dépendent d'hébergeurs géolocalisés**,
tissant un réseau de communication le plus vaste au-travers de plusieurs opérateurs dans plusieurs pays.
Je ne pense évidemment pas que tout le monde doit s'auto-héberger,
mais je pense que **DAVANTAGE** de monde doit sortir des Big Mailer Corps pour aller n'importe où ailleurs…
il faut réintroduire de la contrainte chez les les Big Mailer Corps,
qu'ils soient obligés d'interagir avec des opérateurs en dehors de leur gang,
et qu'il y ait un risque financier **SI** soudainement ils prenaient des décisions à l'encontre des populations.

Les Big Mailer Corps ne sont pas mauvais ou méchants,
**j'utilise moi-même leurs services régulièrement**,
et la plupart de leurs employés cherchent probablement le bien commun…
mais ces entreprises sont devenues trop grosses et puissantes,
il faut qu'il y ait un équilibre parce que l'on ne sait pas comment elles évolueront dans les années à venir,
on ne sait pas non plus comment évoluera la politique de leurs pays d'origine dans les années à venir,
et les informations récentes ne décrivent pas un avenir tout rose dans la bonne direction.

Je concluerai en vous recommandant de regarder cette **excellente** présentation de Bert Hubert ([@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert)) de PowerDNS,
à propos de problèmes similaires qui sont en train d'émerger avec le protocole DNS et les craintes qui en découlent autour de la surveillance.
De très nombreux points de sa présentation sont tout aussi valides en ce qui concerne les services de mail.

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/pjin3nv8jAo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

