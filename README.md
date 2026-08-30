# Kom Bouffe

Yooo, petit site créé pour la kom bouffe, l'objectif c'est de tout centraliser et automatiser pour choisir des recettes de saison, consulter le calendrier des fruits/légumes, chercher par ingrédient ou par nom, et proposer de nouvelles recettes.
Le site est hébergé gratuitement sur GitHub Pages et va chercher ses données dans un Google Sheet, via un petit script (Google Apps Script) qui joue le rôle de  API.

## 1. Les fichiers

- **`index.html`** — le site en entier. Une seule page, un seul fichier, fait en HTML + CSS + JavaScript, pas de build, pas de dépendances à installer.
  C'est le seul fichier à uploader sur GitHub pour mettre à jour le site.
- **Le Google Sheet** — la base de données. Onglets utilisés :
  - `BD_Bouffe` — les ingrédients de chaque recette
  - `BD_Instructions` — les instructions de chaque recette
  - `BD_Recette_Saison` — pour chaque recette, les mois où elle est recommandée (calculé automatiquement à partir des légumes de saison)
  - `BD_Calendrier` — le calendrier des fruits/légumes/céréales par mois
  - `Recettes_En_Attente` — les recettes soumises depuis le site, en attente de validation (ne peuvent être validées que par les membres de la kom bouffe, pour éviter d'avoir des recettes de caca au pipi ou de mètre de picon picon brune)
- **Le script Apps Script** — collé dans l'éditeur Apps Script du Google Sheet (menu *Extensions > Apps Script*), il contient tout le code existant (menus, formulaire "Ajouter une recette" dans le tableur, etc.) + les fonctions `doGet` et `doPost` qui exposent l'API web.

## 2. Comment le site récupère les données

Au chargement, `index.html` fait un `fetch()` vers l'URL de l'API (une constante `API_URL` tout en haut du `<script>`) :

```js
const API_URL = 'https://script.google.com/macros/s/XXXXX/exec';
```

Cette URL appelle `doGet()` dans le script, qui lit le Google Sheet et renvoie un JSON avec :
- la liste des recettes (nom, ingrédients, instructions, mois recommandés)
- le calendrier de saison (légumes/fruits/céréales par mois)

Le site construit ensuite tout l'affichage à partir de ce JSON — il n'y a plus aucune donnée écrite en dur dans le HTML.

## 3. Comment une nouvelle recette est ajoutée

1. Quelqu'un remplit le formulaire dans l'onglet "Ajouter une recette" du site et clique sur "Enregistrer".
2. Le site envoie ces données en `POST` vers la même `API_URL`, qui déclenche `doPost()` dans le script.
3. `doPost()` écrit une nouvelle ligne dans l'onglet **`Recettes_En_Attente`** du Google Sheet, avec le statut "En attente".
4. Une personne avec accès au Sheet relit la recette et change le statut en **"Valider"** (liste déroulante en colonne A) — un déclencheur (`onEdit`) bascule alors automatiquement la recette vers `BD_Bouffe` et `BD_Instructions`, qui la rend visible sur le site.

Rien n'est donc publié automatiquement sans validation manuelle (on connaît les barmans...)

## 4. Mettre à jour le site (ce qui va arriver le plus souvent)

Le site n'a pas de "build" : modifier `index.html`, puis le réuploader sur GitHub, suffit.

1. Ouvrir `index.html`, faire les modifications (HTML/CSS/JS dans le même    fichier).
2. Aller sur le dépôt GitHub du projet.
3. *Add file > Upload files*, déposer le nouveau `index.html` — GitHub propose de remplacer l'ancien automatiquement.
4. *Commit changes*.
5. Le site se met à jour tout seul en quelques minutes à l'adresse du site

## 5. Modifier le backend (Apps Script)

Si on doit changer la façon dont les données sont lues ou écrites (`doGet`/`doPost`), ou toute autre fonction du script :

1. Ouvrir le Google Sheet > *Extensions > Apps Script*.
2. Modifier le code.
3. **Important** : modifier le code ne suffit pas. Il faut republier une nouvelle version du déploiement pour que l'URL existante (`API_URL`) serve le nouveau code :
   *Déployer > Gérer les déploiements > crayon d'édition > Version : "Nouvelle version" > Déployer.*
4. L'URL ne change pas, donc rien à toucher côté site.

## 6. Repères de style (pour rester cohérent visuellement)

Toutes les couleurs sont définies une seule fois, tout en haut du `<style>`, sous forme de variables CSS :

```css
:root{
  --ink: #471818;        /* fond général (bordeaux) */
  --ink-light: #2B2620;  /* fond légèrement plus clair, bordures */
  --paper: #F4EFE1;      /* crème — fond des cartes/tableaux */
  --paper-dim: #E9E2CE;  /* crème plus foncé — en-têtes de tableau */
  --orange: #D81616;     /* accent — boutons, titres de section */
  --plum: #3A1010;       /* accent secondaire */
  --charcoal: #1A1512;   /* texte sur fond clair */
}
```

Changer une couleur ici la change donc partout dans le site.

Polices utilisées (chargées depuis Google Fonts, tout en haut du `<head>`) :
- **UnifrakturMaguntia** — le titre "Kom Bouffe", la police K-Fêt quoi
- **Anton** — les titres de recettes
- **Karla** — le texte courant
- **Space Mono** — les labels, boutons, unités (police à chasse fixe)

## 7. Structure de la page (onglets)

Le site utilise un système d'onglets simple : un menu (`<div class="nav">`) avec des boutons `data-view="..."`, et une section `<div class="view" id="view-...">`par onglet. Un clic sur un bouton du menu affiche la section correspondante et cache les autres (fonction `renderAll()` / gestion de `currentView` en JavaScript).

Onglets actuels :
- `view-month` — choisir une recette par mois (tampons ronds). S'ouvre par défaut sur le **mois en cours** (`new Date().getMonth()`), pas sur un mois fixe.
- `view-recipesearch` — chercher une recette par son nom (résultats en direct pendant la frappe)
- `view-season` — calendrier des fruits/légumes de saison (tableaux avec fond crème, en-tête plus foncé, séparateurs verticaux seulement)
- `view-search` — chercher une recette par ingrédient (résultats en direct pendant la frappe)
- `view-add` — formulaire d'ajout d'une recette

Pour ajouter un nouvel onglet : ajouter un bouton dans `.nav`, une nouvelle `<div class="view" id="view-xxx">`, et le code JS pour la remplir.

## 8. Affichage mobile

En dessous de 600px de large (`@media (max-width: 600px)`), l'affichage change :

- Les titres, labels de section, tampons de mois et titres de recette sont agrandis pour rester lisibles.
- Le menu horizontal à 5 boutons est remplacé par un **menu déroulant** : un bouton unique (`#navToggle`) affiche le nom de l'onglet actif ; un clic dessus déroule la liste complète des onglets juste en dessous (à l'intérieur de `#navWrap`, positionné en `absolute` par rapport à ce conteneur — pas par rapport à toute la page, sinon le menu s'ouvre en bas de la page au lieu de sous le bouton).

**Important** : la balise `<meta name="viewport" content="width=device-width, initial-scale=1">` dans le `<head>` est indispensable pour que ces règles mobile se déclenchent réellement sur un téléphone. Sans elle, les navigateurs mobiles simulent un écran large et aucune media query ne s'applique.

## 9. Choses à surveiller

- Le site dépend entièrement de la disponibilité de l'API Apps Script. Si le Google Sheet est renommé, si un onglet est supprimé/renommé, ou si le déploiement est désactivé, le site affichera "Impossible de charger les recettes".
- Il n'y a pas d'authentification sur l'API : n'importe qui connaissant l'URL peut lire les recettes ou soumettre un ajout (mais pas modifier `BD_Bouffe` directement — tout passe par la validation manuelle).
- Le calendrier de saison affiché vient de `BD_Calendrier` ; s'il est vide pour un mois, le site affiche simplement une colonne vide (pas de message d'erreur particulier).
