---
title: "Implementer un appel système pour OpenBSD"
date: 2023-07-25 23:22:00 +0200
authors:
 - "gilles"
categories:
 - technologie
tags:
 - kernel
 - C
 - OpenBSD
---

{{< tldr >}}
J'ai retrouvé une copie imprimée d'un devoir que j'ai eu à faire en 2005,
quand j'étais encore étudiant,
pour implémenter un appel système sous OpenBSD et Linux.
J'ai perdu le fichier source LaTeX alors j'ai décidé de le retaper pour en avoir une copie numérique.
À l'origine,
l'article couvrait les "loadable kernel modules" (LKM) qui ne font plus partie d'OpenBSD,
j'ai donc supprimé cette partie.
J'ai aussi supprimé la partie Linux parce qu'elle ne m'intéressait pas à l'époque,
que  j'ai écrit le minimum vital pour mon devoir et que ce n'est donc pas très intéressant à lire ;-)
{{< /tldr >}}


# Avertissement

Le contenu de cet article est basé sur un devoir que j'ai rédigé en 2005,
lorsque j'étais encore étudiant à Epitech,
donc les choses ont probablement changé et ce n'est pas un tutoriel:
s'il vous plait,
n'écrivez pas d'appels systèmes et ne les soumettez pas à OpenBSD...
sauf si les développeurs d'OpenBSD vous ont demandé de le faire et vous ont dit que c'était une bonne idée.

JE LE RÉPÈTE:
N'... IMPLÉMENTEZ... PAS... D'... APPELS... SYSTÈMES... À... PARTIR... DE... CET... ARTICLE !

L'article a pour objectif de partager de la connaissance,
et d'éviter de perdre quelque chose que j'ai écrit et qui n'existe plus qu'en version papier.
Voyez le comme un vestige archéologique.

J'ai accidentellement leaké une version brouillon de cet article il y a trois ans,
devenue très rapidement populaire suite à un partage sur un site connu...
avant que je ne la supprime parce que je n'avais pas fini de le nettoyer et de mettre cet avertissement.
Cette fois ci,
j'ai été plus malin,
j'ai tout rédigé avant de faire mon commit.

J'ai fait quelques changements mineurs pour aider à la compréhension,
mais je n'ai pas modifié le style d'écriture,
vous lisez le moi d'il y a 18 ans qui faisait un devoir scolaire.

Les exemples sont très simples,
ils ne sont pas là pour servir de fondation à vos projets,
mais juste pour aider certains à faire leurs premiers pas sur les marches de la compréhension.
Trouvez les erreurs est un petit cadeau que je vous laisse:
je vous encourage à commenter ou [soumettre des pull requests](https://github.com/poolpOrg/poolp.org/tree/master/content/posts/2023-07_03/index.md) pour améliorer cet article.


# Quelques rappels nécessaires
## Programme et Processus
Les gens utilisent souvent les deux mots de manière interchangeable mais il est important de comprendre la différence entre un programme et un processus,
particulièrement parce que le même programme peut être autorisé à utiliser un appel système dans un processus mais pas dans un autre **(utilisateur privilégié ou non)**,
mais aussi parce que la notion de processus fait partie de l'API des appels sytème 
**(l'interface des appels système prends un pointeur sur une `struct proc` qui représente un processus)**.

Un programme est un exécutable qui contient un jeu d'instructions qui sont supposées être exécutées et faire quelque chose.
Il réside dans un fichier structuré **(a.out, elf, ...)** dans le système de fichier qui impose des restrictions sur qui peut ou non l'exécuter **(permissions et ownership dans un système de fichiers)**.
Un processus est une instance de ce programme,
qui tourne dans un espace mémoire qui lui est propre,
avec ses propres privilèges.

Si l'on prends `/bin/ls`,
c'est un programme qui liste les répertoires et fichiers.
Lorsqu'un utilisateur l'exécute,
un processus est créé qui va effectivement exécuter ce programme avec les privilèges de l'utilisateur,
dans un espace mémoire qui n'est pas partagé avec les autres processus.

## Kernel et userland
Les systèmes Unix-like ont une architecture où le code est exécuté dans deux zones principales:
le kernel (noyau) et le userland (espace utilisateur).

Le kernel est en charge de **fournir et limiter l'accès aux matériels**,
**imposer des restrictions sur ce qu'un programme peut faire**,
et **fournir un espace mémoire virtuel aux programmes** dans lesquels ils peuvent s'exécutér.

Un programme **s'exécute en userland** et effectue des opérations sur **une mémoire qui lui est allouée par le kernel durant l'initialisation du processus**.
Lorsque le programme doit accéder à un matériel ou a besoin que le kernel fasse une opération qu'il n'est pas autorisé à faire lui même,
**il demande au kernel de déclencher un appel système**.
L'appel système est **une fonction qui fait partie du kernel** et qui s'exécute en lui de la part du processus.


## Appel système
**Un appel système est un service fourni par le kernel pour qu'un processus userland puisse demander au kernel de faire quelque chose à sa place, généralement quelque chose que le programme useland n'est pas capable ou pas autorisé à faire lui même**.

Du point de vue d'un programme,
c'est une fonction un peu particulière qu'il peut appeler comme n'importe quelle autre fonction,
mais qui **ne s'exécute pas dans l'espace mémoire du processus**.
Un programme connait seulement l'interface des appels systèmes mais n'a pas accès à leur implémentation,
il peut donc les appeler,
leur passer des paramètres,
obtenir un résultat,
mais ne peut pas inspecter ce qu'il se passe à l'intérieur de l'appel système pendant qu'il s'exécute.
Il ne peut pas le debugger.

Ça vient avec des effets de bord.
Côté performances,
un appel système bascule de l'exécution en userland à une exécution en kernel qui est plus coûteuse.
Ensuite,
des bugs dans un appel système ont un impact bien différents des bugs dans une fonction:
**une corruption mémoire peut provoquer le crash d'un processus, la même corruption mémoire dans un appel système peut provoquer le crash du système**.

Il y a deux aspects aux appels systèmes:

L'implémentation de l'appel systène,
le code effectif de l'appel système qui sera exécuté dans le kernel lorsqu'il sera appellé,
et l'interface de l'appel système,
la manière dont l'appel système est supposé être appelé depuis une application en userland.

C'est important de différencier les deux comme,
dans OpenBSD,
le prototype pour l'implémentation d'un appel système ne corresponds pas au prototype de l'interface pour le même appel système comme nous allons voir.


{{< note author="gpt-4" >}}
Les appels système servent de passerelle entre les applications utilisateur et le noyau du système d'exploitation de bas niveau. Ils sont une partie intégrale de l'infrastructure d'un système d'exploitation qui fournit un accès contrôlé aux ressources matérielles, gère les processus et gère les interactions avec le système de fichiers, parmi de nombreuses autres tâches.

Alors que les systèmes d'exploitation sont livrés avec un ensemble standard d'appels système, il peut y avoir des cas où vous souhaitez introduire des appels système personnalisés. Ceux-ci pourraient être destinés à du matériel spécialisé, à des exigences de gestion de processus uniques, ou à d'autres personnalisations au niveau de l'OS qui ne sont pas fournies par les appels système intégrés.

Comprendre comment ajouter de nouveaux appels système dans un système comme OpenBSD, ouvre donc une porte vers des innovations et des personnalisations au niveau du système.
{{< /note >}}


# Implémenter des appels système pour OpenBSD
## Pré-requis
### Utiliser un compte privilégié
Il est évident qu'un compte non privilégié n'est pas autorisé à modifier le kernel comme il vérifie les permissions sur le système.
Pour cette raison,
il faut nécessairement avoir un compte privilégié au moins pour installer le kernel modifié.


### Avoir les sources du système
Les sources du système sont disponibles directemnet depuis le projet OpenBSD.
Pour ce devoir,
nous aurons besoin des archives suivantes:

- src.tar.gz
- srcsys.tar.gz

Il faudra les extraire à la racine du système:

```
% doas tar -C / -zxf src.tar.gz
% doas tar -C / -zxf srcsys.tar.gz
```

**(edit: `doas` remplace maintenant `sudo`)**

### Savoir comment rebuilder un kernel
Une fois que vous avez accès aux sources du système,
vous pouvez rebuilder un kernel en utilisant les commandes suivantes:
```
# cd /usr/src/sys/arch/amd64/config
# config GENERIC
# cd ../compile/GENERIC
# make clean depend install
```

Le rebuild ne prends que quelques minutes et une copie de sauvegarde du précedent kernel est faite automatiquement au cas où le nouveau kernel serait instable.


### Rebuilder le système
Rebuilder le système peut être nécessaire si les changements du kernel affectent des outils userland..
Ce peut être le cas par exemple si vous modifiez la `struct proc` qui est utilisée par des outils comme `ps`, `top` ou `uname`.
Rebuilder est aussi simple que de taper:
```
# cd /usr/src
# make build
```

Rebuilder le système est considérablement plus long que pour le kernel et peut prendre de quelques minutes à plusieurs heures selon l'architecture.


## Appel système sans paramètres: `sys_goodbye()`
Pour débuter,
nous allons implémenter l'appel système `sys_goodbye()` qui ne prends aucun paramètre.
Son prototype est:

```c
int
goodbye(void);
```

### Implémentation
```c
#include <sys/types.h>
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/proc.h>
#include <sys/mount.h>
#include <sys/syscallargs.h>

/* displays "Goodbye, cruel world !" on the console */
sys_goodbye(struct proc *p, void *v, register_t *retval)
{
	printf("Goodbye, cruel world !\n");
	return (0);
}
```

### Description
Notre premier appel système affiche la phrase "Goodbye, cruel world !" sur la console.

Il nous permet de voir que **le prototype de l'appel système diffère entre le userland et le kernel**.
OpenBSD fournit **une API unique pour tous les appels systèmes, peu importe le prototype qu'ils exposent au userland**.

Les headers inclus ici sont le set minimal requis pour que l'API des appels système fonctionne normalement.
Certains peuvent paraitre inutilisés par notre fonction mais seront utilisé lors du build pour la plomberie interne du kernel.
L'appel système ne se limite pas seulement à son implémentation,
certains éléments seront ajoutés par la suite indirectement et automatiquement comme on le verra.

Ne premier appel système va ignorer ses paramètres (`struct proc *`, `void *v` et `register_t *retval`),
utiliser `printf()` et retourner 0 pour indiquer à l'appelant que l'exécution s'est bien déroulée.

Ici, `printf()` ne doit pas être confondu avec le `printf()` userland,
le premier est utilisé pour écrire sur la console et non pas sur la sortie standard.


## Appel système avec paramètres: `sys_showparams()`
Notre second appel système,
`sys_showparams()`,
prends un `int` en paramètre et affiche sa valeur sur la console.
Son prototype est le suivant:

```c
int
showparams(int val);
```

### Implementation
```c
#include <sys/types.h>
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/proc.h>
#include <sys/mount.h>
#include <sys/syscallargs.h>

/* displays value of integer parameter to console */
sys_showparams(struct proc *p, void *v, register_t *retval)
{
	struct sys_showparams_args /* {
		syscallarg(int)		val;
	} */ *uap = v;

	printf("showparams(%d)\n", SCARG(uap, val));
	return (0);
}
```

### Description
Contrairement à la précédente,
cette fonction n'ignore pas ses paramètres mais en **extraie un paramètre entier passé dans l'interface userland**.

Pour le faire,
l'appel système déclare un pointeur vers une `struct sys_showparams_args` et le fait pointer sur son deuxième paramètre,
`void *v`.
Il devient clair que ce second paramètre représente d'une certaine manière les paramètres que le userland passe à l'appel système.

La définition de `struct sys_showparams_args` ne fait pas partie de notre implémentation parce qu'elle est générée automatiquement au moment du build.
Chacun de ses champs correspond à un paramètre dans l'interface userland et la macro `SCARG()` permet de déréférencer la structure correctement,
sans avoir à s'inquiéter de l'alignement ou de l'endian de l'architecture.


## Appel système retournant une valeur: `sys_retparam()`
L'appel système `sys_retparam()` prends un paramètre de type `int` et le retourne s'il est inférieur ou égal à 1024, et dans le cas contraire retourne -1, en assignant `EINVAL` à `errno`.
Son prototype est similaire à celui de `sys_showparams()`:

```c
int
retparam(int val);
```

### Implementation
```c
#include <sys/types.h>
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/proc.h>
#include <sys/mount.h>
#include <sys/syscallargs.h>

/* returns value of integer parameter if lesser or equal to 1024 */
sys_retparam(struct proc *p, void *v, register_t *retval)
{
	struct sys_retparam_args /* {
		syscallarg(int)		val;
	} */ *uap = v;
	unsigned int val;

	val = SCARG(uap, val);
	if (val > 1024)
		return (EINVAL);

	*retval = val;

	return (0);
}
```

### Description
les choses sont un petit peu plus complèxes et nous allons devoir plonger sur ce qu'il se passe en dehors de la fonction pour comprendre son fonctionnement.

Le problème est le suivant:
si nous devons retourner 0 en cas de succès et une valeur positive en cas d'erreur,
alors comment faire pour avoir un appel système qui retourne une valeur positive en cas de succès ?

La solution réside dans **le troisième paramètre de notre appel système**.

**La valeur retournée par notre appel système n'est pas la valeur retournée par l'interface de l'appel système pour le userland.** La valeur de retour de notre appel système sert simplement **à déterminer si sont exécution s'est déroulée correctement ou assigner `errno`**. La valeur retournée par l'interface userland est en réalité placée dans le troisième paramètre de notre implémentation d'appel système qui est en rélaité **un tableau de deux registres**.

**Le premier élément de ce tableau représente EAX**,
**il est initialisé à 0 par l'API des appels systèmes avant que notre implémentation ne soit appellée et ne puisse le modifier**. Le second élément est rarement utilisé: il sert à resoudre le cas particulier de `fork()` qui.. retourne **deux valeurs**,
**une pour le processus parent et une pour le processus enfant**.

{{< note author="gilles" >}}
L'article a été écrit pour amd64 où les registres EAX et EDX sont utilisés pour retval,
ce n'est évidemment pas le cas pour toutes les autres architectures.

Pour une meilleur compréhension,
un coup d'oeil a `/usr/src/sys/amd64/amd64/trap.c` (en remplaçant la plateforme par une autre) est utile:
il prépare les paramètres en respectant la convention d'appel,
déclenche l'interruption d'appel système,
fait correspondre les valeurs de registres et d'errno aux bonnes structures pour au final donner une impression uniforme au userland sur toutes les architectures.
{{< /note >}}


## System call poking into struct proc: `sys_retpid()`
Notre dernier appel système,
`sys_retpid()`,
prends un `int` en paramètre qui va faire retourner à notre fonction le pid du processus si la valeur est 0,
le pid du processus parent si la valeur est 1 et échouer en assignant `EINVAL` a `errno` dans les autres cas.
Son prototype est le suivant:

```c
int
retpid(int val);
```

### Implémentation
```c
#include <sys/types.h>
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/proc.h>
#include <sys/mount.h>
#include <sys/syscallargs.h>

/*
 * returns current pid if val == 0
 * returns parent pid if val == 1
 * return -1 and sets errno to EINVAL otherwise
 */
sys_retpid(struct proc *p, void *v, register_t *retval)
{
	struct sys_retpid_args /* {
		syscallarg(int)		val;
	} */ *uap = v;
	unsigned int val;

	val = SCARG(uap, val);
	if (val != 0 && val != 1)
		return (EINVAL);

	if (val == 0)
		*retval = p->p_pid;
	else
		*retval = p->p_pptr->p_pid;

	return (0);
}
```
{{< note author="gilles" >}}
Notez que cette fonction peu ou non exploser,
pour de multiples raisons,
et si vous ne comprenez pas pourquoi,
ne copiez-collez pas ce code optimiste qui ne fait pas les vérifications et le locking adéquats.

Ce commentaire est valide pour toutes les fonctions de cet article.
{{< /note >}}

### Description
Ce dernier appel nous permet d'illustrer que la fonction ne s'exécute pas en userland mais vraiment dans le kernel,
**il nous permets d'accéder à la mémoire en dehors de celle du processus courant**.
Ici,
nous déréférençons la `struct proc` associée à notre processus mais aussi un pointeur vers une autre `struct proc`,
nous pouvons utiliser les différentes listes chainées internes à la `struct proc` pour accéder à des ressources **qui ne sont pas accessibles à notre processus courant en userland**.

Notez que c'est juste un exemple et qu'il faut faire bien attention à faire le bon locking requis,
**si l'appel système accède à une ressource qui a été libérée, le résultat ne sera pas un crash du processus mais un crash du système**. 


### Intégration
{{<note author="gilles">}}
La version initiale de cette article date de 2005 et présentait le link static et les loadable kernel modules.
**Depuis, l'interface LKM a été supprimée d'OpenBSD, j'ai donc supprimé les parties qui ne présentaient plus d'intérêt de nos jours** pour garder l'article court.
{{</note>}}


{{< note author="gpt-4" >}}
Lors de la mise en œuvre de nouveaux appels système, il est essentiel de faire de la sécurité une priorité dans vos considérations. Par conception, les appels système font le pont entre l'espace utilisateur et le noyau, ce qui, s'il n'est pas correctement géré, peut exposer le système à diverses vulnérabilités.

Lors de la conception d'un appel système, il est crucial de valider toutes les données en entrée. Comme les appels système fonctionnent avec des privilèges de niveau noyau, toutes les données en entrée peuvent potentiellement interagir avec des parties critiques du système, et doivent donc être soigneusement examinées.

Réfléchissez attentivement aux capacités que votre nouvel appel système devrait avoir. Si un appel système a seulement besoin de lire des données, il ne devrait pas avoir la capacité d'écrire des données. Limiter la fonctionnalité au minimum nécessaire peut limiter les dommages potentiels si l'appel système est mal utilisé.

De plus, les problèmes de concurrence peuvent conduire à des conditions de course dans les appels système. Des primitives de synchronisation appropriées doivent être utilisées pour éviter ces scénarios. N'oubliez pas, une faille dans un appel système peut mettre en péril la sécurité de l'ensemble du système, donc être attentif aux vulnérabilités potentielles est de la plus haute importance.

Gardez à l'esprit qu'OpenBSD, comme d'autres systèmes de type Unix, suit le principe du privilège minimal, qui suggère qu'un processus ne devrait se voir accorder que les privilèges qui sont essentiels à sa fonction. En tant que concepteur d'appel système, votre responsabilité est de vous assurer que votre appel système est en accord avec ce principe.
{{< /note >}}



#### Intégration d'appels système par linkage statique
Plusieurs fichiers sont en jeu lors d'une intégration statique d'appel système:

- `/usr/src/sys/kern.syscalls.master` est le fichier principal utilisé pour ajouter un appel système. Il est utilisé pour rebuilder un ensemble de tableaux et de structures internes utilisées par l'API des appels systèmes.
- `/usr/src/sys/kern/syscalls.c` contient la liste des appels système.
- `/usr/src/sys/kern/init_sysent.c` contient la table `sysent`. Chaque élément de la table décrit le nombre de paramètres de l'appel système, la structure qui est associée à ces paramètres et la fonction utilisée pour implémenter l'appel système.
- `/usr/src/sys/sys/syscallargs.h` contient la définition des structures associées aux appels système.
- `/usr/src/sys/sys/syscall.h` contient les numéros associés à nos appels systèmes représentés par des macros.

La première étape est d'éditer `/usr/src/sys/kern.syscalls.master` et de trouver un numéro inutilisé d'appel système,
ou d'en ajouter un s'il n'y en a pas de disponible.
Le format du fichier est très simple,
il consiste en un numéro d'appel système, un type d'appel système et un pseudo-prototype.

Une fois modifié,
les fichiers autogénérés peuvent être reconstruits par ces commandes:

```
# cd /usr/src/sys/kern
# make init_sysent.c
```

Les fichiers décrits ci-dessus sont reconstruits en prenant en compte les nouveaux appels systèmes et en produisant les structures nécessaires pour leurs paramètres.
Tout ce qu'il reste à faire est de reconstruire le kernel après avoir ajouté les fichiers contenant les implémentations de nos appels système.

Les appels système sont indépendant de l'architecture,
leurs implémentations sont placéees dans `/usr/src/sys/kern`.

Vous devez ensuite éditer `/usr/src/sys/kern/files` et y ajouter les lignes suivantes:
```
file kern/sys_goodbye.c
file kern/sys_showparam.c
file kern/sys_retparam.c
file kern/sys_retpid.c
```
puis reconstruire le kernel.

À ce stade,
**une fois le système redémarré sur le nouveau kernel**,
nos appels systèmes sont utilisables par les applications en userland
**qui connaissent leur numéro**
au travers de l'appel système `syscall()`.

Pour pouvoir les appeler par leur nom,
**vous devrez mettre à jour les fichiers d'entêtes** `/usr/include/sys/syscall.h` et `/usr/include/sys/sycallargs.h` avec ceux générés durant la phase de `make init_sysent.c`,
**puis reconstruire la libc après avoir ajouté les fichiers objets** (sans leur préfixe sys_) dans `/usr/src/lib/libc/sys/Makefile.inc`.

La libc est reconstruire avec les commandes suivantes:
```
# cd /usr/src/lib/libc
# make install
```

Les appels systèmes sont immédiatement disponibles sans besoin d'un redémarrage du système.


# Un mot du futur, 2023
Soyez gentils avec le `gilles@` de 2005,
n'hésitez pas à commenter ou soumettre une pull request pour mettre à jour l'article et le moderniser.

Au cas où vous vous le demandez,
cette interface n'est pas exactement celle de NetBSD ou de FreeBSD,
mais les interfaces et structures se ressemblent assez pour pouvoir démarrer.
Si vous le voulez,
je pourrais écrire quelques articles autour de ces sujets à l'avenir.

Au cas où vous vous le demandez aussi,
cette interface ne vous aidera pas du tout pour écrire un appel système sous Linux.
Tout diffère,
depuis la manière dont le code est organisé,
à la manière dont les appels systèmes sont enregistrés,
en passant par la convention d'appel.
Les concepts sont les mêmes,
l'implémentation et les API sont radicalement différentes.
J'ai écrit un devoir là dessus aussi,
implémentant _exactement_ les mêmes appels systèmes en décrivant le cheminement,
mais j'ai trouvé ça pénible au possible et ça se ressentait dans l'article :laughing:


# Lectures en rapport...

Je recommande la lecture de ces deux livres sur le sujet:

| | |
|---|---|
| <iframe sandbox="allow-popups allow-scripts allow-modals allow-forms allow-same-origin" style="width:120px;height:240px;" marginwidth="0" marginheight="0" scrolling="no" frameborder="0" src="//ws-eu.amazon-adsystem.com/widgets/q?ServiceVersion=20070822&OneJS=1&Operation=GetAdHtml&MarketPlace=FR&source=ss&ref=as_ss_li_til&ad_type=product_link&tracking_id=poolporg07-21&language=fr_FR&marketplace=amazon&region=FR&placement=B00O56CFEE&asins=B00O56CFEE&linkId=a08afca27d9380d55d64d3ca0474379c&show_border=true&link_opens_in_new_window=true"></iframe> | <iframe sandbox="allow-popups allow-scripts allow-modals allow-forms allow-same-origin" style="width:120px;height:240px;" marginwidth="0" marginheight="0" scrolling="no" frameborder="0" src="//ws-eu.amazon-adsystem.com/widgets/q?ServiceVersion=20070822&OneJS=1&Operation=GetAdHtml&MarketPlace=FR&source=ss&ref=as_ss_li_til&ad_type=product_link&tracking_id=poolporg07-21&language=fr_FR&marketplace=amazon&region=FR&placement=B002MZAR6I&asins=B002MZAR6I&linkId=f8c0f0b545e506479897b17244cc5c62&show_border=true&link_opens_in_new_window=true"></iframe> |