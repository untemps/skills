---
name: solve-issue
description: >
  Traite une issue GitHub de bout en bout de façon autonome : récupération du contenu,
  analyse, plan d'action soumis à validation, implémentation sur une branche dédiée
  avec commits atomiques, validation (tests + build + lint), mise à jour de l'environnement
  de démo et de la documentation, puis push et création de la PR vers main.
  Déclenche ce skill dès que l'utilisateur mentionne une issue GitHub à traiter, résoudre,
  implémenter ou corriger — que ce soit via "/solve-issue <id>", "traite l'issue 42",
  "fix issue #123", "implémente la feature de l'issue 7", ou toute formulation similaire.
---

# Traitement d'une issue GitHub

Ce workflow prend en charge une issue GitHub de la récupération à la PR, en passant par
l'implémentation validée. Le principe directeur : avancer vite mais proprement, avec des
commits qui racontent une histoire claire et un humain dans la boucle avant de toucher au code.

## 1. Récupérer et analyser l'issue

```bash
gh issue view <id> --json number,title,body,labels,assignees,milestone,state
```

À partir des données récupérées :

- **Détermine le type** à partir des labels GitHub en priorité, puis des mots-clés du titre :
  - `bug`, `fix`, `regression` → type `fix`
  - `enhancement`, `feature`, `feat` → type `feat`
  - `documentation`, `docs` → type `docs`
  - `chore`, `refactor`, `ci`, `test` → type correspondant
  - Sans signal clair → demande à l'utilisateur

- **Comprends l'intention** : lis le body en entier. Si des détails sont ambigus ou manquants
  (comportement attendu peu clair, scope incertain, dépendances non mentionnées), pose une
  question ciblée avant de rédiger le plan — mieux vaut clarifier maintenant que retravailler
  après implémentation.

## 2. Préparer et soumettre le plan

Rédige un plan structuré et présente-le à l'utilisateur **avant toute modification du code**.

### Format du plan

```
## Plan — Issue #<id> : <titre>

**Type** : fix | feat | docs | ...
**Branche** : <type>/<id>-<slug-du-titre>

### Approche
<2-4 phrases décrivant la stratégie technique>

### Changements prévus
- `src/foo.ts` — <ce qui change et pourquoi>
- `src/bar.ts` — <ce qui change et pourquoi>
- ...

### Tests
- <test à ajouter ou modifier>
- ...

### Environnement de démo
<comment la démo sera mise à jour : vérification que ça tourne toujours / ajout d'un exemple>

### Documentation
<fichiers à mettre à jour, ou "aucune modification nécessaire">

### ⚠️ Breaking changes potentiels
<liste des breaking changes, ou "aucun">

### Validation prévue
<scripts de vérification identifiés dans package.json qui tourneront après chaque commit>

### Commits prévus
1. `<type>(<scope>): <description>` — <contenu>
2. ...
```

Attends la validation explicite de l'utilisateur. Si des ajustements sont demandés, mets à
jour le plan et resoumets-le avant de continuer.

## 3. Créer la branche

Une fois le plan validé :

```bash
git checkout main && git pull
git checkout -b <type>/<id>-<slug>
```

Le slug est dérivé du titre de l'issue : minuscules, mots significatifs séparés par des
tirets, 3-5 mots max. Exemples : `fix/42-tooltip-overflow`, `feat/7-keyboard-navigation`.

## 4. Implémenter et valider

Pour chaque commit prévu dans le plan :

### a) Implémenter le changement

Fais un seul changement cohérent à la fois. Un commit = une unité de sens (une correction,
une nouvelle fonction, un refactor ciblé). Ne regroupe pas des changements sans rapport.

### b) Valider

Avant de valider, lis les scripts disponibles dans `package.json` pour savoir exactement
ce qui existe dans ce projet. Lance ensuite tous les scripts de vérification pertinents,
dans un ordre logique :

1. **Tests** — script CI de préférence (`test:ci`, `test`, `vitest run`, etc.)
2. **Build** — compilation et bundling (`build`, `build:lib`, etc.)
3. **Tooling** — tous les scripts de qualité présents : lint, format check, type check,
   et tout autre script qui vérifie la cohérence du code (`lint`, `check`, `typecheck`,
   `tsc`, `format:check`, `validate`, etc.)

L'objectif est de ne laisser passer aucune régression détectable par les outils du projet.
Si un script de vérification existe dans `package.json`, il doit tourner vert.

Si une étape échoue, **ne commite pas et ne continue pas**. Analyse l'erreur, corrige-la
dans le même commit ou dans un commit de fix dédié, puis relance la validation complète.

### c) Commiter

```bash
git add <fichiers concernés>
git commit -m "<type>(<scope>): <description>"
```

Utilise les Conventional Commits. Le `<scope>` est optionnel mais utile quand il apporte
de la clarté. La description doit toujours commencer par un verbe à l'infinitif en anglais
(ex. `fix(tooltip): Correct overflow on mobile`, `feat(api): Add pagination support`).

## 5. Environnement de démo

Après avoir complété l'implémentation principale, mets à jour l'environnement de démo
(généralement dans `demo/`, `playground/`, `examples/`, ou similaire) :

- **Pour un bug fix** : vérifie que la démo tourne toujours (`yarn dev` dans le dossier
  concerné, pas d'erreur au démarrage) et que le bug corrigé n'est plus reproductible.
- **Pour une feature** : ajoute un exemple concret qui illustre la nouvelle fonctionnalité
  de façon à ce qu'un développeur puisse la tester visuellement.

Commite les changements de démo séparément si substantiels :

```bash
git commit -m "demo: <description de ce qui a été ajouté/modifié>"
```

## 6. Documentation

Passe en revue :
- `README.md` : API publique, options, exemples de code
- Tout fichier `.md` qui documente les fonctionnalités touchées
- Les commentaires JSDoc/TSDoc sur les fonctions et types modifiés

Mets à jour ce qui est obsolète ou incomplet. Si la feature ajoute une nouvelle option ou
modifie une API existante, la documentation doit le refléter.

```bash
git commit -m "docs: update documentation for <feature/fix>"
```

Si aucune documentation n'est à modifier, passe cette étape.

## 7. Push et création de la PR

### Pousser la branche

```bash
git push -u origin <branche>
```

### Créer la PR

> **Langue** : tous les éléments textuels de la PR (titre, résumé, descriptions des changements, plan de test, breaking changes) doivent être rédigés **en anglais**.

```bash
gh pr create \
  --base main \
  --title "<type>: <titre de l'issue>" \
  --body "$(cat <<'EOF'
## Résumé
<2-3 phrases décrivant ce qui a été fait>

## Changements
- <changement 1>
- <changement 2>

## Plan de test
- [ ] <étape de test 1>
- [ ] <étape de test 2>

Closes #<id>
EOF
)"
```

### Breaking changes

Si l'implémentation introduit un breaking change (modification d'API publique, changement
de comportement observable, suppression d'une option), ajoute une section explicite dans
le body de la PR :

```markdown
## ⚠️ Breaking changes
- **`<élément>`** : <description précise de ce qui change et de l'impact pour les utilisateurs>
- Migration : <ce que les utilisateurs doivent faire pour s'adapter>
```

Et utilise le préfixe `!` dans le titre du commit concerné :
`feat!: rename option X to Y` ou `fix!: change default behavior of Z`.

### Retour à l'utilisateur

Une fois la PR créée, fournis :
- L'URL de la PR
- Un résumé de ce qui a été implémenté
- La liste des commits créés
- Un rappel si des breaking changes ont été introduits
