---
name: update-deps
description: >
  Workflow complet pour mettre à jour les dépendances d'un projet Node.js (npm, yarn, pnpm, bun).
  Déclenche ce skill dès que l'utilisateur mentionne une mise à jour de dépendances, un bump de versions,
  "update deps", "mettre à jour les paquets", "upgrader les dépendances", ou toute variante.
  Utilise ce skill même si le projet n'est pas encore identifié — le skill commence par analyser
  l'environnement. Couvre : détection du gestionnaire de paquets, analyse des breaking changes,
  mise à jour progressive avec commits atomiques, validation (tests + build + lint), environnement
  de démo, et mise à jour de la documentation.
---

# Mise à jour des dépendances

Ce workflow met à jour les dépendances d'un projet Node.js de façon progressive et sûre, en
informant l'utilisateur à chaque étape risquée et en maintenant un historique git propre.

## 1. Préparer l'environnement

### Détecter le gestionnaire de paquets

Cherche dans cet ordre :
- `bun.lockb` → bun
- `yarn.lock` → yarn
- `pnpm-lock.yaml` → pnpm
- `package-lock.json` → npm
- Sinon, regarde le champ `packageManager` dans `package.json`

Note : certains monorepos ont un gestionnaire racine différent des sous-packages. Adapte-toi.

### Identifier les commandes du projet

Lis `package.json` (et ceux des sous-packages/workspaces si pertinent) pour trouver les scripts disponibles. En général :
- `test` ou `test:ci` → suite de tests
- `build` → compilation/bundling
- `lint` ou `check` → linting/formatage
- `dev` → serveur de développement (utile pour vérifier l'env de démo)

Si ces scripts n'existent pas avec ces noms exacts, détecte les équivalents en lisant les scripts disponibles.

### Créer la branche

```bash
git checkout -b chore/update-deps
```

Si la branche existe déjà, demande à l'utilisateur s'il veut la réutiliser ou repartir de zéro depuis main/master.

## 2. Inventorier les dépendances

Liste toutes les dépendances avec leur version actuelle et la dernière version disponible :

```bash
# Yarn
yarn outdated

# npm
npm outdated

# pnpm
pnpm outdated

# bun
bun outdated
```

Classe les dépendances en deux catégories :
- **`dependencies` + `devDependencies`** du projet principal
- **Environnement de démo** (sous-dossier `demo/`, `playground/`, `examples/`, ou tout dossier avec son propre `package.json` qui n'est pas un workspace)

Présente un tableau récapitulatif à l'utilisateur avant de commencer :

```
Dépendance       Actuelle   Dernière   Saut majeur ?
---------------- ---------- ---------- -------------
react            17.0.2     19.1.0     oui (×2)
vite             4.5.0      6.2.1      oui
typescript       5.3.0      5.4.5      non
...
```

## 3. Analyser les breaking changes

Pour chaque dépendance qui nécessite un saut de version majeure, recherche les breaking changes **avant** de procéder :

- Lis le CHANGELOG ou les release notes sur GitHub/npm
- Utilise `npm info <package> homepage` ou `npm info <package> repository` pour trouver les sources
- Cherche les mentions de `BREAKING CHANGE`, `migration guide`, `deprecated`

Pour chaque dépendance à risque, informe l'utilisateur clairement :

> ⚠️ **react** 17 → 19 : plusieurs breaking changes majeurs.
> - Suppression des event pools synthétiques
> - `ReactDOM.render` remplacé par `createRoot`
> - Nouvelles règles hooks
> Veux-tu tenter la mise à jour (les tests t'alerteront si quelque chose casse) ou la reporter ?

Attend la confirmation pour les dépendances à risque identifié avant de les inclure dans le plan.

## 4. Grouper et ordonner les mises à jour

Regroupe les dépendances de façon logique pour que chaque commit ait du sens :
- Les outils de build ensemble (vite, rollup, esbuild…)
- Les outils de test ensemble (vitest, jest, testing-library…)
- Les outils de qualité ensemble (eslint, prettier, typescript…)
- Les dépendances liées à un même framework ensemble
- Les dépendances sans lien, mises à jour mineures/patch : peuvent être groupées

Les mises à jour risquées (saut majeur avec breaking changes identifiés) méritent leur propre commit.

Présente le plan de mise à jour groupé à l'utilisateur et attends sa validation avant de commencer.

## 5. Mettre à jour et valider

Pour chaque groupe :

### a) Mettre à jour les versions

Modifie `package.json` directement ou utilise la commande du gestionnaire :

```bash
# Exemple yarn
yarn add <package>@latest

# Pour mettre à jour vers une version spécifique
yarn add <package>@<version>
```

Pour les dépendances de développement, utilise le flag approprié (`--dev` / `-D`).

### b) Installer

```bash
yarn install   # ou npm install, pnpm install, bun install
```

### c) Valider

Lance dans l'ordre :
1. Les tests : `yarn test:ci` (ou équivalent)
2. Le build : `yarn build`
3. Le linting : `yarn lint` (ou `yarn check`)

Si une étape échoue, **ne continue pas**. Montre l'erreur à l'utilisateur et demande :
> La commande `yarn test:ci` a échoué après la mise à jour de `<package>`. Veux-tu :
> 1. Que je tente de corriger l'erreur
> 2. Revenir en arrière sur cette mise à jour et passer à la suite
> 3. Mettre en pause pour investiguer toi-même

Attends la réponse avant de continuer.

### d) Commiter

Si tout passe, crée un commit :

```bash
git add package.json yarn.lock  # (ou le lockfile approprié)
git commit -m "chore(deps): update <groupe>"
```

Exemples de messages :
- `chore(deps): update build tools (vite 4 → 6, rollup 3 → 4)`
- `chore(deps): bump typescript 5.3 → 5.4`
- `chore(deps): update testing-library and vitest`

## 6. Environnement de démo

Après avoir traité les dépendances principales, cherche un environnement de démo :
- Sous-dossier nommé `demo/`, `playground/`, `examples/`, `app/`, ou similaire
- Présence d'un `package.json` indépendant (pas un workspace géré par le root)
- Présence d'un script `dev` dans ce `package.json`

Si trouvé, applique le même processus (inventaire → analyse → mise à jour → validation → commit), avec des commits préfixés par exemple `chore(demo): update deps`.

À la fin, **vérifie que l'environnement de démo démarre correctement** :

```bash
cd demo && yarn dev &   # lance en background
sleep 5
# Vérifie qu'il n'y a pas d'erreur de démarrage dans le output
kill %1
```

Si le démarrage échoue, signale-le à l'utilisateur.

## 7. Mettre à jour la documentation

Passe en revue :
- `README.md` : versions mentionnées, badges, instructions d'installation, prérequis
- Tout autre fichier `.md` dans le projet qui mentionne des numéros de version ou des commandes
- Les commentaires dans le code qui font référence à des versions spécifiques de dépendances

Mets à jour automatiquement les références obsolètes et inclus ces changements dans un commit dédié si nécessaire :

```bash
git commit -m "docs: update version references after deps upgrade"
```

Ne touche pas à la documentation si aucune référence n'est obsolète.

## 8. Résumé final

Présente un résumé des changements effectués :

```
✅ Mises à jour appliquées :
  - vite 4 → 6 (build tools)
  - vitest 0.34 → 3.1 (testing)
  - typescript 5.3 → 5.4 (minor)

⏭️  Reportées (à ta demande) :
  - react 17 → 19 (breaking changes importants)

⚠️  Échecs / non appliquées :
  - some-package 2 → 3 (tests cassés, rollback effectué)

📦 Environnement de démo : mis à jour et fonctionnel
📝 Documentation : mise à jour
```

Rappelle à l'utilisateur de passer en revue les commits et de merger la branche quand il est satisfait.
