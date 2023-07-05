---
title: "fatigue et envie de vacances..."
date: 2005-05-21 00:00:00
category: ["myLife"]
authors:
 - "gilles"
draft: false
categories:
 - technology
---

Je sais pas pourquoi, mais je suis epuise alors que j'ai dormi pres de 14h cette nuit.
J'ai envie de faire une pause,
de me prendre une annee de vacances au bord de la plage les doigts de pieds dans le sable fin blanc pendant qu'on me sert du sorbet a la noix de coco sur fond de Gojira alors que le soleil me rechauffe le visage mais juste un peu, pas trop, voila, comme ca, merci.
L'odeur des langoustes grilles me chatouille les narines et, alors que je sirote une pinte de Guinness(c) (la premire d'une longue serie ?),
j'entends le doux murmure des vagues _wuuuuuush .... wuuuuuuush ..._ couvert par le cri des mouettes _t'as un syscall a finiiiiir ... t'as un syscall a finiiiiiiiiir ..._ 

En rouvrant les yeux, un peu triste et du, j'entrevois deux lignes ... 

```shell
$ cc -c -I/usr/src/sys -D_KERNEL sys_whore.c 
$ 
```

... qui me ramene  la dure realite... j'ai un syscall a terminer. 

Meme dans mes songes, je passe vraiment des vacances de merde ... 

[Ajout] 
Bon alors voila,
j'ai perdu plusieurs heures dessus donc je vais donner une astuce qui va peut-etre epargner de nombreuses heures de recherche a mes camarades tech4 qui codent leur syscall pour BSD. 

```c
int
sys_whore(struct proc *p, void *v, register_t *retval)
{
  return (0);
}
```

Mais a quoi peut bien servir retval alors qu'on a un return() dans le code,
et comment faire pour faire retourner autre chose qu'un entier a notre syscall ? 

Apres m'etre cogne pres d'une dizaine de syscalls sans en voir un seul qui utilisait le retval,
j'ai demande sur le channel #netbsd ou on m'a dit de la merde.

Un peu plus tard sur #openbsd,
on m'a conseill la lecture de /usr/src/sys/i386/i386/trap.c et la tout devient evident a la lecture de la fonction syscall(). 

En realite,
que se passe-t'il lorsqu'un programme demande l'execution de notre syscall ? 
- Les parametres sont places en ordre inverse dans la stack 
- Le numro de syscall est place dans le registre EAX 
- L'interruption 0x80 fait passer en mode kernel qui execute... la fonction syscall() definie dans le trap.c 

Quand on regarde ce qu'elle fait, on comprends mieux. Quelques morceaux choisis: 

```c
    register_t code, args[8], rval[2];
    [...]
    rval[0] = 0;  /* valeur de retour de l'appel systeme */
    [...]

    /* appel; on stock le return */
    orig_error = error = (*callp->sy_call)(p, args, rval);
    [...]

    switch (error) {
    [...]
    default:  /* return different de zero (et de cas particuliers) */
    bad:
    if (p->p_emul->e_errno)
	error = p->p_emul->e_errno[error];
        frame.tf_eax = error;
        frame.tf_eflags |= PSL_C;       /* carry bit */
        break;
    }
```

Explications: 
Le premier element de rval est la valeur de retour de notre syscall, a ne pas confondre avec le return.
En fait, le return dans le code de notre syscall sert *uniquement* a determiner s'il s'est bien deroule,
il renvoie 0 en cas de succes *quoi qu'il arrive*,
et une valeur d'errno dans le cas contraire.
Mais alors ... c'est pour ca que retval n'est presque jamais utilise dans les syscalls ! 

En effet,
si le syscall rate son return sera un code errno et la fonction wrapper n'aura pas besoin de verifier retval puisque la valeur qu'elle devra renvoyer sera -1 ou NULL, donc pas besoin d'y toucher. De meme, si syscall reussi, comme generalement il est sense renvoyer 0, il n'a pas besoin d'y toucher non plus puisque retval[0] est initialise a 0 avant le call. Le seul cas ou retval est modifie alors c'est quand le syscall renvoie une valeur differente de 0 en cas de success...
la lecture de `getpid()` et de ses soeurs le confirme...
YOUPI ! 

Ca vous rappelle rien ?
Dans ktrace en tech3 on avait eu le meme probleme,
lorsqu'un syscall ratait il renvoyait non pas -1 mais une valeur differente de 0 et il fallait ruser avec EAX pour determiner s'il s'agissait d'un ratage ou pas pour afficher nous meme -1 comme etant la valeur de retour. 

Un mystere qui m'a occupe une partie de la journe est resolu ;) 

[Ajout #2] 
Si vous vous demandiez pourquoi retval est un tableau de deux entiers,
la reponse m'est apparue tout a l'heure, je sais pas, un clair de lucidite.
Comme son nom l'indique, c'est un tableau de registre (register_t),
et si le premier element correspond au registre EAX (bah oui puisque la valeur de retour est placee dans ce registre),
le fait que syscall() associe le registre EDX au second element peut sembler un peu bizarre. 

Et bien nan,
en fait c'est pour resoudre un cas bien particulier,
celui de sys_fork() puisqu'il renvoie DEUX valeurs de retour en cas de succes.
Le pere recupere la valeur de retour; le pid, dans EAX et le fils recupere la valeur de retour, 0, dans EDX.
Magique non ?
