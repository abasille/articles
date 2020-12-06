# Maîtriser la réécriture de l'historique Git

## Pourquoi réécrire l'historique Git ?

Le processus d'écriture du code informatique n'est pas linéaire :

- On se trompe → Correction.
- On enrichi → Ajout.
- On refactorise ou optimise → Réécriture.
- On identifie du code mort ou on renonce à une modification → Suppression.

Ces différentes opérations, dont la liste n'est clairement pas exhaustive se produisent au grès de la réflexion du développeur durant les phases de codage. Elles ne sont pas toutes prédictibles (car potentiellement absentes de la conception précédant le codage).

Certaines étapes méritent d'être communiquées aux autres membres de l'équipe car elles permettent de comprendre la raison de la modification. Elles doivent être des étapes importantes de la construction du code et doivent apparaitre dans l'historique Git.

D'autres, non, car elles n'apportent aucune information. Par exemple, si vous avez renommé X fois une variable avant de trouver son nom définitif, seul le dernier renommage est signifiant et lui seul devrait apparaître dans l'historique Git de la branche.

> “Le récit n'est plus l'écriture d'une aventure, mais l'aventure d'une écriture.”
>
> -- Jean Ricardou - Pour une théorie du nouveau roman

> Trouver une citation sur le storytelling

La réécriture de l'historique Git doit amener du storytelling dans `git log` afin d'améliorer la compréhension de l'état actuel du code en ne dévoilant **QUE** les étapes significatives de construction. Il doit permettre à chacun de répondre à la question "_Pourquoi ce code a t-il été écrit comme ça ?_" lors de la revue d'une PR ou bien tout simplement lors de l'activité d'analyse du code quotidienne de tout développeur.

### Un exemple

Prenons l'exemple suivant : Votre projet importe une librairie tierce non maintenue qui provoque un bug sur certains navigateurs. Décision est prise de la remplacer par une librairie équivalente. Le développeur assigné projète de suivre les étapes suivantes :

1. CFG - Installation et configuration de la nouvelle librairie dans le projet.
2. MIG - Migration de toutes les références à la précédente librairie vers la nouvelle.
3. SUP - Désinstallation de l'ancienne librairie.

La chronologie semble évidente à suivre. Mais que se passe t-il dans la réalité ?

1. CFG
2. MIG-1
3. REF-1: Durant l'étape MIG, il a découvert du vieux code ne respectant pas nouvelles bonnes pratiques qui ont évoluées au sein de l'équipe. Il réécrit ce code qui n'a rien à voir avec la librairie à migrer.
4. SUP
5. REF-2: Il relit son code et n'est pas satisfait du nommage d'une variable introduite par l'étape REF-1. Il la renomme.
6. MIG-2: Il lance les tests qui échouent car il a oublié, dans l'étape MIG-1, des appels à l'ancienne librairie. Il corrige.

Maintenant, mettez-vous à la place du reviewer et considérez les 2 cas suivants :

- Le développeur n'a fait qu'un seul commit dans sa PR : Le reviewer doit identifier lui-même à quelle étape CFG, MIG ou SUP le code modifié se rapporte. De plus du code, qui n'a rien à voir avec la migration a été modifié (REF-1 & REF-2). Celui-ci aurait dû faire l'objet d'une PR séparée.
- 1 commit par étape : Le reviewer, qui aime lire chaque commit, perd du temps avec les commit MIG-2 et REF-2 qui auraient dûes être inclues respectivement dans MIG-1 et REF-1.

Idéalement, le développeur aurait dû faire 3 commits Étapes dans sa PR (CFG, MIG-1/MIG-2 et SUP) et ouvrir une autre PR contenant un seul commit regroupant REF-1 + REF-2.

Comment aurait-il pu réécrire l'historique Git de sa branche ? C'est ce que nous allons voir maintenant.

## Où réécrire l'historique Git ?

Lorsque vous travaillez seul à un instant _t_ sur une branche, sentez-vous en complète confiance pour réécrire l'historique Git de votre branche.

Ainsi, si vous devez passer la main à un collègue, celui-ci n'aura qu'à exécuter un simple `git pull --rebase` s'il possédait une précédente version de la branche en local.

## Où l'éviter ?

### Sur une branche de référence

Le risque d'une réécriture de l'historique Git sur une branche duquelle dérive d'autres branches (ex : `main`) est que ses branches dérivées ne soient plus basées sur un commit de `main`. S'il est techniquement possible de les rebaser sur un nouveau commit, cela ajoute inutilement une surcharge de travail.

De plus, si chaque PR ne répond qu'à un seul objectif (versus un amalgame de plusieurs fonctionnalités, correctifs ou optimisations) et que chaque PR est mergée avec la stratégie `squash` dans la branche de référence, il n'y plus beaucoup de raisons valables pour réécriture l'historique Git d'une branche de référence.

### Sur une branche partagée entre plusieurs développeurs

Selon le nombre de développeurs travaillant sur une branche, leur niveau d'aisance avec Git et la capacité de l'équipe à communiquer, il est parfois souhaitable d'éviter la réécriture de l'historique Git sur une branche partagée.

Le risque est de "perdre" du code mais surtout du temps (résolution de conflits de rebase), lorsqu'un développeur réécrit l'historique sans prévenir préalablement ses collègues.

Une solution rapide et efficace est d'attendre la fin des développements sur cette branche pour laisser le soin au développeur le plus à l'aise avec Git de réécrire l'historique avant de créer la PR.

## 8 cas d'usage

Sauf mention contraire, chacun des cas d'usage suivants :

- suppose que `origin/main` est la branche d'où dérive la branche de développement courante.
- opère une réécriture de l'historique Git nécessitant l'utilisation de la commande suivante :
  `git push --force-with-lease`
  afin d'enregistrer les modifications sur le repository distant. Dans le cas contraire, le *push* sera rejeté par le serveur Git. 

### Répartir les modifications d'un fichier dans plusieurs commit

**Quand ?** Les modifications effectuées sur un fichier sont de natures différentes et méritent d'être réparties dans plusieurs commits *Étapes*.

1. Ajouter ou supprimer les lignes modifiées dans la _Staging Area_ :
   `git add --patch` ou `git remove --patch` puis la commande `edit`
2. Commiter le contenu de la *Staging Area* :
   `git commit -m "Description du commit Étape"`
3. S'il reste des modifications, revenir à l'étape 1.

Je ne détaillerai pas les subtilités de ces commandes car elles ne sont généralement pas utilisées directement par les développeurs. En effet, avec VSCode, ce sont les actions équivalentes sont plus véloces : "*> Git: Stage Selected Ranges*" et "*> Git: Unstage Selected Ranges*" pour respectivement ajouter ou retirer une sélection de texte à la *Staging Area*.

**Nota** : Pas de réécriture de l'historique Git (`git push` peut être appelé sans l'option `--force-with-lease`).

**Démo**

- FIX Reformulation texte détectée par hazard
- FEAT La fonctionnalité initialement prévue

### Ajouter des modifications au dernier commit

**Quand ?** Les modifications non commitées aurients dûes être incluses dans le dernier commit.

La commande suivante ajoute le contenu de la *Staging Area* au dernier commit :

`git commit --amend`

Action VSCode équivalente : "*> Git: Commit Staged (Amend)*"

### Ajouter du code dans un commit précédent

**Quand ?** Les modifications non commitées auraient dûes être incluses dans un précédent commit.

- Identifiez dans l'historique Git, le commit dans lequel ajouter les modifications. Les formats de commit ID courts (ex : `6faa6ad`) ou long (ex : `a8304f46e7d759e0c70ffc2e65409add2e34dda4`) sont tout deux utilisables pour la suite :
   `git log`
  VSCode : Extension *Source Control* > *Commits*
- Déclarez le contenu de la *Staging Area* comme _fixant_ le commit cible `<COMMIT_ID>` :
  `git commit --fixup=<COMMIT_ID>`
- Fusionnez le commit *fixant* avec le commit cible avec un rebasage intéractif :
  `git rebase -i origin/main`.
  L'éditeur défini par défaut pour Git s'ouvre avec la liste des commits de la branche courante. Remarquez que votre dernier commit _fixant_ a été déplacé juste après le commit cible `6faa6ad` et qu'il possède l'opération `fixup`. Quittez l'éditeur pour valider la fusion des deux commits.

### Fusionner des commits

**Quand ?** Lorsqu'une même étape significative est dispatchée dans plusieurs commits.

- Lancez un rebasage intéractif :
  `git rebase -i origin/main` 
  L'éditeur défini par défaut pour Git s'ouvre avec la liste des commits de la branche courante.
- Déplacer le ou les commits à fusionner immédiatement après le commit *Étape* cible et appliquez leurs l'une des deux commandes suivantes :
  - `squash` : Fusion avec réécriture du message du commit cible.
  - `fixup` : Fusion sans réécriture du message commit cible. 

### Scinder un commit

**Quand ?** Les modifications effectuées dans un commit appartiennent à plusieurs Étapes significatives et devraient être réparties dans de nouveaux commits ou fusionnées avec des commits existants.

- Lancez un rebasage intéractif :
  `git rebase -i origin/main` 
  L'éditeur défini par défaut pour Git s'ouvre avec la liste des commits de la branche courante.
- Remplacer `pick` par la commande `edit` devant le numéro du commit à scinder. Enregistrez le fichier et quittez l'édition afin de commencer le rebasage.
  Le rebasage sur la branche `origin/main` est exécuté jusqu'au commit associé à la commande `edit`, puis s'arrête.
- Supprimez le commit à scinder et transférez toutes ses modifications dans la *Staging Area* :
  `git reset --soft HEAD~`
- Ne gardez dans la Staging Area que les modifications d'une même Étape Significative en retirant un ou plusieurs fichiers de la Staging Area ou en appliquant la technique du § "*Répartir les modifications d'un fichier dans plusieurs commit*". Puis commitez le contenu de la Staging Area :
  `git commit -m "Description de l'Étape Significative"`
  Recommencez tant que la *Staging Area* n'est pas vide afin de créer autant de commits que vous avez identifiez d'*Étapes Significatives*.
- Terminez le rebasage :
  `git rebase --continue`

### Supprimer un commit

**Quand ?** Les modifications effectuées dans un commit n'ont pas vocation à apparaître dans la PR et doivent être définitivement supprimées.

- Lancez un rebasage intéractif :
  `git rebase -i origin/main` 
  L'éditeur défini par défaut pour Git s'ouvre avec la liste des commits de la branche courante.
- Remplacer `pick` par la commande `drop` devant le numéro du ou des commits à supprimer. Enregistrez le fichier et quittez l'édition afin de terminer le rebasage.

### Ordonner les commits

**Quand ?**  Pour faciliter la revue de la PR en définissant la chronologie la plus pertinente possible ou bien en vue de manipuler un ensemble de commits afin de les extraires vers une autre branche.

- Lancez un rebasage intéractif :
  `git rebase -i origin/main` 
  L'éditeur défini par défaut pour Git s'ouvre avec la liste des commits de la branche courante.
- Permuttez les commits.
  Enregistrez le fichier et quittez l'édition afin de valider le nouvel ordonnancement des commits.

### Déplacer des commits vers une autre branche

**Quand ?** Lorsque les modifications effectuées dans un ou plusieurs commits :

- ont vocations à être validées dans une PR dédiée.

- sont nécessaires à l'avancée des développements sur une autre branche.

Depuis la branche dans laquelle intégrer les commits :

- Ajouter le commits :
  `git cherry-pick <COMMIT_ID>`
- Ajouter un interval de commit :
  `git cherry-pick <COMMIT_FIRST_ID>..<COMMIT_LAST_ID>`
  Nota : Deux points ("..") sans espace entre les deux versions de commit.

**Nota** : Pas de réécriture de l'historique Git (`git push` peut être appelé sans l'option `--force-with-lease`).

## Résumé des commandes Git abordées

| Commandes Git                                                | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `git add --patch`<br />`git remove --patch`                  | Ajouter ou supprimer les lignes modifiées dans la _Staging Area_ |
| `git commit --fixup` <br />+ `git rebase -i <BRANCH>`        | Ajouter du code dans un commit précédent                     |
| `git commit --amend`                                         | Ajouter du code au commit précédent                          |
| `git rebase -i` <BRANCH><br />+ commande `squash`            | Fusion AVEC réécriture du message du commit cible            |
| `git rebase -i` <BRANCH><br />+ commande `fixup`             | Fusion SANS réécriture du message du commit cible            |
| `git rebase -i` <BRANCH><br />+ commande `edit`<br />+ `git reset --soft HEAD~`<br />+ `git commit -m "..."`<br />+ `git rebase --continue` | Scinder un commit                                            |
| `git rebase -i` <BRANCH><br />+ commande `drop`              | Supprimer un commit.                                         |
| `git cherry-pick C1..Cn`                                     | Déplacer l'interval de commits C1 à Cn inclus vers une autre branche |
| `git pull --rebase`                                          | Récupérer locallement la branche distance dont l'historique a été réécrit. |
| `git push --force-with-lease`                                | Enregistrer les modifications sur le repository distant suite à une réécriture de l'hostorique. |

