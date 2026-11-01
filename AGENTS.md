# Mockup — mode d’emploi pour agents IA / guide for AI agents

> Français d’abord, English below. Même contenu en HTML : `agent/index.html`. Résumé : `llms.txt`.

<!-- lang:fr -->

Ce document s’adresse aux agents IA (Claude Code, Codex, ou tout agent qui pilote un navigateur via Playwright / CDP). Il explique comment utiliser **Mockup** (`app.html`), le générateur de mockups réalistes de Binks Design : importer une photo, laisser l’app calculer la profondeur, poser un logo, choisir une matière, régler, puis exporter un PNG, un JPEG, la carte de profondeur ou le projet `.json`.

Il existe en deux endroits, toujours identiques : `AGENTS.md` (Markdown, français puis anglais) et la page `agent/index.html`. Résumé court : `llms.txt`.

Points d’entrée découvrables sans JavaScript : `index.html` et `docs.html` déclarent `<link rel="alternate" type="text/markdown" href="AGENTS.md">`, `<link rel="help" href="agent/">` et `<link rel="alternate" type="text/plain" href="llms.txt">`, et contiennent un commentaire HTML et un `<noscript>` qui renvoient ici. `robots.txt` autorise tout.

## Règle fondamentale

Mockup pose sur une photo le logo que l’utilisateur fournit. Ce n’est **pas** un générateur de logos ni de photos.

- **N’invente jamais** un logo, un symbole, une couleur de marque, une police ou un texte de marque. Le fichier logo (PNG, SVG, WEBP) et la photo fournis par l’utilisateur **font foi**.
- Ne redessine pas un SVG fourni, ne le vectorise pas, ne modifie pas sa géométrie ni ses couleurs. Ne remplace pas un fichier manquant par un fichier « approchant » : demande-le.
- **Recolorer le logo** change la couleur de tout le logo : ne l’active que si l’utilisateur le demande ou si sa charte le prévoit.
- Les démos (t-shirt, bouteille, logo de la marque fictive **AVORA**) servent à tester. Ne livre jamais un mockup avec le logo AVORA à la place du logo du client.
- **Aucune clé API.** L’app n’en utilise aucune et n’en demande aucune : n’en demande jamais à l’utilisateur.
- **Ta photo reste locale.** Le calcul de profondeur (Depth Anything V2 Small) et la sélection d’objet (SlimSAM) tournent dans le navigateur avec Transformers.js. Seuls la bibliothèque et les modèles publics sont téléchargés (jsDelivr, Hugging Face). Aucune image n’est envoyée à un serveur.

Après chaque action, relis l’état (textes de statut, fichier téléchargé, capture). **Un clic n’est pas une preuve de réussite.**

## Démarrer

### Servir et ouvrir l’app

Le dépôt est un site statique : `index.html` (accueil), `app.html` (l’application, un seul fichier), `docs.html` (documentation), `agent/index.html` (cette page), `AGENTS.md`, `llms.txt`, `robots.txt`, `assets/` (pages uniquement).

```bash
python3 -m http.server 8000
# puis ouvrir http://127.0.0.1:8000/app.html
```

- **http:// recommandé** : le cache des modèles (Cache Storage) et le Web Worker de profondeur ne fonctionnent qu’en http(s). En `file://`, l’app marche mais recalcule sans cache.
- **WebGL2 obligatoire.** En Chromium headless, lance par exemple avec `--use-angle=swiftshader --enable-unsafe-swiftshader` pour avoir WebGL2 logiciel. Sans WebGL2, l’app affiche « Ce navigateur ne prend pas en charge WebGL2 » et rien ne marche.
- **Internet la première fois** : polices, icônes (Lucide), Transformers.js et modèles. Selon le réseau, huggingface.co peut refuser un navigateur automatisé : l’app passe alors seule à l’**estimation rapide** (voir § 8).
- Prêt quand : `document.title === "Mockup · Binks Design"` et le bouton `#pickPhoto` est visible.

### Deux façons de piloter

1. **Par l’interface** (recommandé) : champs `<input type="file">` cachés, boutons et curseurs avec des `id` et `data-*` stables (§ 4).
2. **Lecture de l’état** : l’app expose un objet global interne `window.__mockup` (§ 3). Il n’est pas documenté comme API publique : utilise-le pour **lire** l’état et attendre, pas pour remplacer l’interface.

## Pas d’API publique : ce qui existe

Mockup n’a **pas** d’API JavaScript dédiée aux agents (pas d’équivalent de `window.binksyGuide`). N’en invente pas et n’en ajoute pas.

L’app expose toutefois, pour son propre débogage, `window.__mockup`. C’est un objet **interne, sans garantie de stabilité**. Parties utiles en lecture :

```js
__mockup.S.photo            // null, ou { W, H, name, src } : photo chargée (W×H = pleine résolution)
__mockup.S.depthSource      // 'none' | 'ai' | 'quick' | 'custom'
__mockup.S.depthBusy        // true pendant le calcul IA
__mockup.S.device           // 'webgpu' | 'wasm' | '' : où a tourné l’IA
__mockup.S.layers           // calques : [{ name, preset, visible, shape: 'plan'|'cyl', corners, ... }]
__mockup.S.sel              // index du calque sélectionné (-1 si aucun)
__mockup.S.mode             // 'transform' | 'persp' | 'warp' | 'mask' | 'pick'
__mockup.S.view2            // { s, ox, oy } : échelle et décalage de la vue (voir § 5)
Object.keys(__mockup.PRESETS) // ids des 13 matières
```

Il contient aussi des fonctions internes (`loadDemo()`, `loadDemoBottle()`, `exportImage('png'|'jpg')`, `undo()`, `redo()`…). Préfère les boutons correspondants ; si tu les appelles, vérifie le résultat dans l’interface.

## Repères stables

Tous ces sélecteurs existent dans `app.html` (version bureau, fenêtre de plus de 900 px de large).

| Sélecteur | Rôle |
|---|---|
| `#fPhoto` | `input[type=file]` photo (JPG, PNG, WEBP) |
| `#fLogo` | `input[type=file]` logos (PNG, SVG, WEBP), `multiple` |
| `#fDepth` | `input[type=file]` carte de profondeur (image) |
| `#fProj` | `input[type=file]` projet `.json` |
| `#pickPhoto` · `#demoBtn` · `#demoBottle` | Écran d’accueil : Choisir une photo · Démo t-shirt · Démo bouteille (visibles seulement sans photo) |
| `#st1` · `#st2` · `#st3` | Étapes Photo / Profondeur / Logo ; textes d’état `#st1s`, `#st2s`, `#st3s` |
| `#addLayer` · `#addLogo2` | Ajouter un logo |
| `#layers .layer[data-i="N"]` | Calque N (clic = sélection) ; `[data-act="vis"]` masquer/afficher, `[data-act="del"]` supprimer |
| `#modes [data-mode="transform\|persp\|warp\|mask"]` | Outils Déplacer / Perspective (Forme en cylindre) / Déformer / Masque |
| `#views [data-view="render\|depth"]` | Vue Rendu / Profondeur |
| `#undo` · `#redo` · `#compare` · `#fit` | Annuler · Rétablir · Avant/après (maintenir) · Ajuster à l’écran |
| `#tabs [data-tab="mat\|depth"]` | Onglets Matière / Profondeur du panneau |
| `#shapeSeg [data-s="plan\|cyl"]` | Forme Libre / Cylindre |
| `#cylDetect` · `#backSw` | Détecter l’objet · Visible au dos (interrupteur, classe `on`) |
| `#chips .chip[data-p="<id>"]` | Matière (classe `on` = active) |
| `#coreSliders .sl[data-k="<clé>"] input` | Curseurs principaux : `disp`, `shade`, `spec`, `tex`, `flou` |
| `#moreBtn` puis `#advSliders .sl[data-k]` | `soft`, `bleed`, `refl`, `frost`, `relief` (−1000 à 1000), `stitch`, `tint`, `burn`, `glass`, `glow` |
| `#opSlider .sl[data-k="opacity"] input` | Opacité |
| `#blendSeg [data-b="0\|1\|2"]` | Normal / Produit / Incrustation |
| `#recolorSw` · `#ink` · `#inkSws button[data-c]` | Recolorer (interrupteur) · couleur · teintes rapides |
| `#bifFold` · `#bifCh [data-c="0..3"]` · `#bifReset` | Options de fusion : déplier · Gris/Rouge/Vert/Bleu · réinitialiser |
| `#fitSurf` · `#resetWarp` | Épouser la surface · Réinitialiser la déformation |
| `#pickBtn` · `#pickKind [data-k="clip\|front"]` · `#pickSign [data-s="1\|-1"]` | Sélection d’objet · Logo sur l’objet / Objet devant · Ajouter / Retirer |
| `#brushSeg [data-e="1\|0"]` · `#brushSliders .sl[data-k="size\|hard"]` | Pinceau Effacer / Restaurer · taille, dureté |
| `#autoSw` · `#autoSlider .sl[data-k="autoSens"]` · `#clearMask` | Masque automatique · sensibilité · réinitialiser le masque |
| `#depthSliders .sl[data-k="smooth\|gain"]` | Lissage · Intensité du relief |
| `#showDepthSw` · `#reDepth` · `#quickDepth` · `#importDepth` · `#exportDepth` | Onglet Profondeur |
| `#exportBtn` puis `#exportPop [data-x="png\|jpg\|depth\|save\|open"]` | Menu Exporter |
| `#toastT` | Texte du dernier message (toast) |
| `#ov` | Canevas des poignées (souris, glisser) |

Ids des matières (`data-p`) : `tissu`, `carton`, `vitre`, `metal`, `papier`, `plastique`, `broderie`, `verre`, `canette`, `ceramique`, `cuir`, `bois`, `neon`. `verre`, `canette` et `ceramique` sont pensées pour la forme Cylindre.

Sous 900 px de large (ou en paysage téléphone), l’app passe en mise en page mobile : barre `#mTabs [data-sheet="photo\|logos\|mat\|fus\|mask"]`, menu `#mMore`, feuilles `#sheet`. Pilote plutôt en fenêtre bureau (par exemple 1440 × 900).

## Parcours pas à pas

### Exemple Playwright (Python)

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    b = p.chromium.launch(args=["--use-angle=swiftshader", "--enable-unsafe-swiftshader"])
    page = b.new_page(viewport={"width": 1440, "height": 900}, accept_downloads=True)
    page.goto("http://127.0.0.1:8000/app.html")
    page.wait_for_selector("#pickPhoto")

    # 1. Photo fournie par l’utilisateur
    page.set_input_files("#fPhoto", "photo-client.jpg")
    # 2. Attendre la profondeur (IA ou estimation rapide)
    page.wait_for_function("__mockup.S.photo && !__mockup.S.depthBusy && __mockup.S.depthSource !== 'none'",
                           timeout=300_000)
    print(page.inner_text("#st2s"))   # « Carte IA prête », « Estimation rapide »…
    # 3. Logo fourni par l’utilisateur (centré, 34 % du petit côté de la photo)
    page.set_input_files("#fLogo", "logo-client.svg")
    page.wait_for_function("__mockup.S.layers.length === 1")
    # 4. Matière
    page.click('#chips [data-p="tissu"]')
    # 5. Export PNG pleine résolution
    page.click("#exportBtn")
    with page.expect_download() as d:
        page.click('#exportPop [data-x="png"]')
    d.value.save_as("photo-client-mockup.png")
    # 6. Projet .json pour reprendre plus tard
    page.click("#exportBtn")
    with page.expect_download() as d:
        page.click('#exportPop [data-x="save"]')
    d.value.save_as("photo-client.mockup.json")
    b.close()
```

### Placer le logo

Le logo arrive centré. Pour le placer :

- **Clavier** : sélectionne le calque, puis flèches (1 px d’écran) ou `Maj` + flèches (10 px).
- **Souris** : glisse dans le logo pour le déplacer ; poignées carrées des coins et des bords pour la taille ; poignée ronde au-dessus pour pivoter (`Maj` : par pas de 15°). Pour viser un point de la photo, convertis ses coordonnées (pixels de la photo) en coordonnées du canevas `#ov` : `x = ix * S.view2.s + S.view2.ox`, `y = iy * S.view2.s + S.view2.oy`, puis ajoute la position de `#ov` (`bounding_box()`).
- **Perspective** (`P`) : glisse les 4 coins. **Déformer** (`D`) : grille de 4 × 4 points. **Épouser la surface** (`#fitSurf`) incline le logo selon la carte de profondeur.
- **Cylindre** (bouteille, canette, tasse) : `#shapeSeg [data-s="cyl"]`, puis `#cylDetect` et un clic sur l’objet ; le cadre se cale (ou, sans IA, un cadre estimé est proposé).

### Régler un curseur

Les curseurs sont des `input[type=range]` de 0 à 1000 (relief : −1000 à 1000), recréés à chaque mise à jour du panneau : recherche-les juste avant de les modifier.

```js
const set = (sel, v) => { const i = document.querySelector(sel); i.value = v;
  i.dispatchEvent(new Event('input', { bubbles: true })); i.dispatchEvent(new Event('change', { bubbles: true })); };
set('#coreSliders .sl[data-k="flou"] input', 0);      // logo net
set('#opSlider .sl[data-k="opacity"] input', 950);    // opacité 95 %
```

`input` met à jour l’aperçu, `change` l’inscrit dans l’historique. Double-clic sur un curseur : retour à la valeur de la matière.

## Exports et fichiers

Tous les fichiers arrivent comme **téléchargements** du navigateur (attrape-les avec `expect_download`). `<photo>` = nom du fichier photo sans extension.

| Menu `#exportPop` | Fichier | Contenu |
|---|---|---|
| `[data-x="png"]` (ou `Ctrl`/`⌘`+`E`) | `<photo>-mockup.png` | Rendu à la résolution de la photo (limite : taille de texture max de la carte graphique ; 4096 px sur écran tactile) |
| `[data-x="jpg"]` | `<photo>-mockup.jpg` | Même rendu, JPEG qualité 0,95 |
| `[data-x="depth"]` | `<photo>-profondeur.png` | Carte de profondeur en niveaux de gris, taille de la photo |
| `[data-x="save"]` (ou `Ctrl`/`⌘`+`S`) | `<photo>.mockup.json` | Projet complet |
| `[data-x="open"]` | — | Ouvre `#fProj` pour charger un projet |

Le projet `.mockup.json` est un JSON `{ app: "binks-mockup", version: 2, photo: { src, name }, depth: { source, w, h, data, device }, depthP, layers: [...] }` : la photo et les logos y sont intégrés (data URL ou texte SVG), avec pour chaque calque position, maillage, matière, réglages, forme et masque. Ouvre-le avec `page.set_input_files("#fProj", "fichier.mockup.json")` ou en le déposant sur la page ; toast « Projet ouvert ».

## État persistant

- **Aucune sauvegarde automatique du travail.** Recharger la page efface photo, logos et réglages. Pour garder un travail : enregistre le projet `.json`.
- `localStorage` (origine de la page) : `mockup.bif` (`"1"` / `"0"` : options de fusion dépliées ou non) et `mockup.modelOK` (`"1"` : l’utilisateur a accepté sur mobile de télécharger le modèle). Rien d’autre.
- Cache Storage : `transformers-cache` (modèles téléchargés par Transformers.js) et `mockup-probe` (test de disponibilité du cache).
- Utilise un profil de navigateur dédié aux tests si tu ne veux pas laisser ces traces.

## Profondeur et IA

- Calcul auto à chaque nouvelle photo. Statut dans `#st2s` : « Chargement du modèle… », « Téléchargement du modèle », « Analyse de la profondeur… », puis « Carte IA prête », « Estimation rapide », « Carte importée » ou « IA indisponible · estimation rapide ».
- `__mockup.S.depthSource` : `ai` (Depth Anything V2), `quick` (estimation rapide, sans IA), `custom` (carte importée). `S.device` : `webgpu` ou `wasm`.
- Le premier calcul IA télécharge le modèle (environ 100 Mo d’après l’app) ; compte plusieurs minutes sur un processeur lent ou un rendu logiciel. Prévois un `timeout` large.
- Si l’IA échoue, l’app bascule seule sur l’estimation rapide : c’est un résultat utilisable, mais **dis-le** à l’utilisateur. `#reDepth` relance l’IA ; `#importDepth` / `#fDepth` accepte une carte fournie (clair = proche).
- La sélection d’objet (`#pickBtn`, `#cylDetect`) charge SlimSAM la première fois ; en cas d’échec : toast « La sélection IA a besoin d’internet la première fois · cadre estimé sans IA ».

## Vérifications avant de rendre le travail

1. Le logo utilisé est **exactement** le fichier fourni (pas AVORA, pas une version redessinée).
2. `#st2s` indique la source de profondeur ; si c’est l’estimation rapide ou une carte importée, signale-le.
3. Le fichier exporté existe, s’ouvre, et a la taille attendue (celle de la photo, sauf limite de la carte graphique).
4. Regarde l’export : logo visible, bien placé, pas coupé par un masque involontaire, pas de recoloration non demandée.
5. Si l’utilisateur doit reprendre le travail, fournis aussi le `.mockup.json`.
6. Aucune erreur dans la console (les avertissements de chargement du modèle IA mis à part, s’ils ont mené à l’estimation rapide).

## Limites et erreurs

| Message (toast `#toastT`) | Cause / quoi faire |
|---|---|
| « Ce navigateur ne prend pas en charge WebGL2 » / « WebGL2 est requis par ce navigateur » | Active WebGL2 (voir § 2) |
| « Importe d’abord une photo » | Ajouter un logo avant la photo |
| « Ce logo n’a pas pu être lu » | Fichier logo invalide : redemande-le, ne le recrée pas |
| « Le modèle IA n’a pas pu se charger · estimation rapide utilisée » | Réseau ou Hugging Face indisponible |
| « La profondeur n’est pas encore prête » | `Épouser la surface` avant la fin du calcul : attends |
| « Ce fichier n’est pas un projet valide » | Le `.json` n’est pas un projet Mockup |
| « L’export a échoué : image trop grande pour ce navigateur » | Photo trop grande pour la carte graphique |

Autres limites : un SVG est rendu en image (jusqu’à 4096 px de côté) ; la profondeur se calcule sur une version de travail de 2048 px maximum ; changer de photo supprime les logos et l’historique.

## Logique à suivre pour chaque tâche

1. Demande (ou vérifie que tu as) la **photo** et le **logo** de l’utilisateur. Sans logo, n’en fabrique pas.
2. Sers le dossier en http, ouvre `app.html` dans une fenêtre bureau avec WebGL2.
3. Importe la photo, attends la profondeur, note sa source.
4. Importe le logo, place-le, choisis la matière adaptée à la surface (Cylindre pour bouteille, canette, tasse).
5. Masque ce qui doit passer devant ; garde le flou à 0 sauf demande.
6. Exporte le PNG (et le `.json` si utile), vérifie (§ 9), puis rends les fichiers en disant ce qui a été réglé.

<!-- lang:en -->

This document is for AI agents (Claude Code, Codex, or any agent driving a browser through Playwright / CDP). It explains how to use **Mockup** (`app.html`), Binks Design’s realistic mockup generator: import a photo, let the app compute depth, place a logo, pick a material, adjust, then export a PNG, a JPEG, the depth map or the `.json` project.

It lives in two places that always say the same thing: `AGENTS.md` (Markdown, French then English) and the `agent/index.html` page. Short summary: `llms.txt`.

Entry points that need no JavaScript: `index.html` and `docs.html` declare `<link rel="alternate" type="text/markdown" href="AGENTS.md">`, `<link rel="help" href="agent/">` and `<link rel="alternate" type="text/plain" href="llms.txt">`, and include an HTML comment and a `<noscript>` pointing here. `robots.txt` allows everything.

## Golden rule

Mockup places the logo the user provides onto a photo. It is **not** a logo or photo generator.

- **Never invent** a logo, symbol, brand colour, font or brand copy. The logo file (PNG, SVG, WEBP) and the photo provided by the user **are the reference**.
- Do not redraw a provided SVG, do not vectorise it, do not change its geometry or colours. Never replace a missing file with a “close enough” one: ask for it.
- **Recolour the logo** changes the colour of the whole logo: only turn it on if the user asks or the brand guidelines say so.
- The demos (t-shirt, bottle, logo of the fictional brand **AVORA**) are for testing. Never deliver a mockup with the AVORA logo instead of the client’s logo.
- **No API key.** The app uses none and asks for none: never ask the user for one.
- **The photo stays local.** Depth estimation (Depth Anything V2 Small) and object selection (SlimSAM) run in the browser with Transformers.js. Only the library and public models are downloaded (jsDelivr, Hugging Face). No image is sent to a server.

After every action, read the state back (status texts, downloaded file, screenshot). **A click is not proof of success.**

## Getting started

### Serve and open the app

The repository is a static site: `index.html` (home), `app.html` (the app, one file), `docs.html` (documentation), `agent/index.html` (this page), `AGENTS.md`, `llms.txt`, `robots.txt`, `assets/` (pages only).

```bash
python3 -m http.server 8000
# then open http://127.0.0.1:8000/app.html
```

- **http:// recommended**: the model cache (Cache Storage) and the depth Web Worker only work over http(s). Over `file://` the app works but recomputes without cache.
- **WebGL2 is required.** In headless Chromium, launch for instance with `--use-angle=swiftshader --enable-unsafe-swiftshader` to get software WebGL2. Without WebGL2 the app shows “Ce navigateur ne prend pas en charge WebGL2” and nothing works.
- **Internet the first time**: fonts, icons (Lucide), Transformers.js and models. Depending on the network, huggingface.co may refuse an automated browser: the app then falls back to the **quick estimate** on its own (see § 8).
- Ready when: `document.title === "Mockup · Binks Design"` and the `#pickPhoto` button is visible.

### Two ways to drive it

1. **Through the UI** (recommended): hidden `<input type="file">` fields, buttons and sliders with stable `id` and `data-*` attributes (§ 4).
2. **Reading state**: the app exposes an internal global object `window.__mockup` (§ 3). It is not documented as a public API: use it to **read** state and to wait, not to replace the UI.

## No public API: what exists

Mockup has **no** JavaScript API meant for agents (no equivalent of `window.binksyGuide`). Do not invent one and do not add one.

For its own debugging, the app does expose `window.__mockup`. It is an **internal object with no stability guarantee**. Useful read-only parts:

```js
__mockup.S.photo            // null, or { W, H, name, src }: loaded photo (W×H = full resolution)
__mockup.S.depthSource      // 'none' | 'ai' | 'quick' | 'custom'
__mockup.S.depthBusy        // true while the AI runs
__mockup.S.device           // 'webgpu' | 'wasm' | '': where the AI ran
__mockup.S.layers           // layers: [{ name, preset, visible, shape: 'plan'|'cyl', corners, ... }]
__mockup.S.sel              // selected layer index (-1 if none)
__mockup.S.mode             // 'transform' | 'persp' | 'warp' | 'mask' | 'pick'
__mockup.S.view2            // { s, ox, oy }: view scale and offset (see § 5)
Object.keys(__mockup.PRESETS) // ids of the 13 materials
```

It also holds internal functions (`loadDemo()`, `loadDemoBottle()`, `exportImage('png'|'jpg')`, `undo()`, `redo()`…). Prefer the matching buttons; if you call them, check the result in the UI.

## Stable hooks

All these selectors exist in `app.html` (desktop layout, window wider than 900 px).

| Selector | Role |
|---|---|
| `#fPhoto` | photo `input[type=file]` (JPG, PNG, WEBP) |
| `#fLogo` | logos `input[type=file]` (PNG, SVG, WEBP), `multiple` |
| `#fDepth` | depth map `input[type=file]` (image) |
| `#fProj` | `.json` project `input[type=file]` |
| `#pickPhoto` · `#demoBtn` · `#demoBottle` | Start screen: Choose a photo · T-shirt demo · Bottle demo (only visible with no photo) |
| `#st1` · `#st2` · `#st3` | Photo / Depth / Logo steps; status texts `#st1s`, `#st2s`, `#st3s` |
| `#addLayer` · `#addLogo2` | Add a logo |
| `#layers .layer[data-i="N"]` | Layer N (click = select); `[data-act="vis"]` hide/show, `[data-act="del"]` delete |
| `#modes [data-mode="transform\|persp\|warp\|mask"]` | Move / Perspective (Shape on a cylinder) / Warp / Mask tools |
| `#views [data-view="render\|depth"]` | Render / Depth view |
| `#undo` · `#redo` · `#compare` · `#fit` | Undo · Redo · Before/after (hold) · Fit to screen |
| `#tabs [data-tab="mat\|depth"]` | Material / Depth panel tabs |
| `#shapeSeg [data-s="plan\|cyl"]` | Free / Cylinder shape |
| `#cylDetect` · `#backSw` | Detect the object · Visible on the back (switch, class `on`) |
| `#chips .chip[data-p="<id>"]` | Material (class `on` = active) |
| `#coreSliders .sl[data-k="<key>"] input` | Main sliders: `disp`, `shade`, `spec`, `tex`, `flou` (blur) |
| `#moreBtn` then `#advSliders .sl[data-k]` | `soft`, `bleed`, `refl`, `frost`, `relief` (−1000 to 1000), `stitch`, `tint`, `burn`, `glass`, `glow` |
| `#opSlider .sl[data-k="opacity"] input` | Opacity |
| `#blendSeg [data-b="0\|1\|2"]` | Normal / Multiply (Produit) / Overlay (Incrustation) |
| `#recolorSw` · `#ink` · `#inkSws button[data-c]` | Recolour (switch) · colour · quick swatches |
| `#bifFold` · `#bifCh [data-c="0..3"]` · `#bifReset` | Blending options: unfold · Grey/Red/Green/Blue · reset |
| `#fitSurf` · `#resetWarp` | Fit to surface · Reset warp |
| `#pickBtn` · `#pickKind [data-k="clip\|front"]` · `#pickSign [data-s="1\|-1"]` | Object selection · Logo on the object / Object in front · Add / Remove |
| `#brushSeg [data-e="1\|0"]` · `#brushSliders .sl[data-k="size\|hard"]` | Brush Erase / Restore · size, hardness |
| `#autoSw` · `#autoSlider .sl[data-k="autoSens"]` · `#clearMask` | Automatic mask · sensitivity · reset mask |
| `#depthSliders .sl[data-k="smooth\|gain"]` | Smoothing · Relief strength |
| `#showDepthSw` · `#reDepth` · `#quickDepth` · `#importDepth` · `#exportDepth` | Depth tab |
| `#exportBtn` then `#exportPop [data-x="png\|jpg\|depth\|save\|open"]` | Export menu |
| `#toastT` | Text of the latest message (toast) |
| `#ov` | Handles canvas (mouse, drag) |

Material ids (`data-p`): `tissu` (fabric), `carton` (cardboard), `vitre` (window), `metal`, `papier` (paper), `plastique` (plastic), `broderie` (embroidery), `verre` (glass), `canette` (can), `ceramique` (ceramic), `cuir` (leather), `bois` (engraved wood), `neon`. `verre`, `canette` and `ceramique` are meant for the Cylinder shape.

Below 900 px wide (or in phone landscape), the app switches to its mobile layout: bar `#mTabs [data-sheet="photo\|logos\|mat\|fus\|mask"]`, menu `#mMore`, sheets `#sheet`. Prefer driving it in a desktop window (for example 1440 × 900).

## Step by step

### Playwright example (Python)

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    b = p.chromium.launch(args=["--use-angle=swiftshader", "--enable-unsafe-swiftshader"])
    page = b.new_page(viewport={"width": 1440, "height": 900}, accept_downloads=True)
    page.goto("http://127.0.0.1:8000/app.html")
    page.wait_for_selector("#pickPhoto")

    # 1. Photo provided by the user
    page.set_input_files("#fPhoto", "client-photo.jpg")
    # 2. Wait for depth (AI or quick estimate)
    page.wait_for_function("__mockup.S.photo && !__mockup.S.depthBusy && __mockup.S.depthSource !== 'none'",
                           timeout=300_000)
    print(page.inner_text("#st2s"))   # "Carte IA prête", "Estimation rapide"…
    # 3. Logo provided by the user (centred, 34 % of the photo's short side)
    page.set_input_files("#fLogo", "client-logo.svg")
    page.wait_for_function("__mockup.S.layers.length === 1")
    # 4. Material
    page.click('#chips [data-p="tissu"]')
    # 5. Full-resolution PNG export
    page.click("#exportBtn")
    with page.expect_download() as d:
        page.click('#exportPop [data-x="png"]')
    d.value.save_as("client-photo-mockup.png")
    # 6. .json project to resume later
    page.click("#exportBtn")
    with page.expect_download() as d:
        page.click('#exportPop [data-x="save"]')
    d.value.save_as("client-photo.mockup.json")
    b.close()
```

### Placing the logo

The logo arrives centred. To place it:

- **Keyboard**: select the layer, then arrows (1 screen px) or `Shift` + arrows (10 px).
- **Mouse**: drag inside the logo to move it; square corner and edge handles to resize; round handle above to rotate (`Shift`: 15° steps). To aim at a point of the photo, convert its photo-pixel coordinates to `#ov` canvas coordinates: `x = ix * S.view2.s + S.view2.ox`, `y = iy * S.view2.s + S.view2.oy`, then add the position of `#ov` (`bounding_box()`).
- **Perspective** (`P`): drag the 4 corners. **Warp** (`D`): 4 × 4 point grid. **Fit to surface** (`#fitSurf`) tilts the logo according to the depth map.
- **Cylinder** (bottle, can, mug): `#shapeSeg [data-s="cyl"]`, then `#cylDetect` and a click on the object; the frame snaps to it (or, without AI, an estimated frame is proposed).

### Setting a slider

Sliders are `input[type=range]` from 0 to 1000 (relief: −1000 to 1000), rebuilt on every panel refresh: query them right before changing them.

```js
const set = (sel, v) => { const i = document.querySelector(sel); i.value = v;
  i.dispatchEvent(new Event('input', { bubbles: true })); i.dispatchEvent(new Event('change', { bubbles: true })); };
set('#coreSliders .sl[data-k="flou"] input', 0);      // sharp logo
set('#opSlider .sl[data-k="opacity"] input', 950);    // 95 % opacity
```

`input` updates the preview, `change` records it in the history. Double-click a slider: back to the material’s value.

## Exports and files

Every file arrives as a browser **download** (catch it with `expect_download`). `<photo>` = photo file name without extension.

| `#exportPop` menu | File | Content |
|---|---|---|
| `[data-x="png"]` (or `Ctrl`/`⌘`+`E`) | `<photo>-mockup.png` | Render at the photo’s resolution (limit: GPU max texture size; 4096 px on touch screens) |
| `[data-x="jpg"]` | `<photo>-mockup.jpg` | Same render, JPEG quality 0.95 |
| `[data-x="depth"]` | `<photo>-profondeur.png` | Greyscale depth map, photo size |
| `[data-x="save"]` (or `Ctrl`/`⌘`+`S`) | `<photo>.mockup.json` | Full project |
| `[data-x="open"]` | — | Opens `#fProj` to load a project |

The `.mockup.json` project is a JSON `{ app: "binks-mockup", version: 2, photo: { src, name }, depth: { source, w, h, data, device }, depthP, layers: [...] }`: photo and logos are embedded (data URL or SVG text), with each layer’s position, mesh, material, settings, shape and mask. Open it with `page.set_input_files("#fProj", "file.mockup.json")` or by dropping it on the page; toast “Projet ouvert”.

## Persistent state

- **No automatic save of the work.** Reloading the page clears photo, logos and settings. To keep work: save the `.json` project.
- `localStorage` (page origin): `mockup.bif` (`"1"` / `"0"`: blending options unfolded or not) and `mockup.modelOK` (`"1"`: on mobile, the user agreed to download the model). Nothing else.
- Cache Storage: `transformers-cache` (models downloaded by Transformers.js) and `mockup-probe` (cache availability test).
- Use a dedicated test browser profile if you do not want to leave these traces.

## Depth and AI

- Runs automatically on every new photo. Status in `#st2s`: “Chargement du modèle…”, “Téléchargement du modèle”, “Analyse de la profondeur…”, then “Carte IA prête”, “Estimation rapide”, “Carte importée” or “IA indisponible · estimation rapide”.
- `__mockup.S.depthSource`: `ai` (Depth Anything V2), `quick` (quick estimate, no AI), `custom` (imported map). `S.device`: `webgpu` or `wasm`.
- The first AI run downloads the model (about 100 MB according to the app); allow several minutes on a slow CPU or software rendering. Use a generous `timeout`.
- If the AI fails, the app falls back to the quick estimate on its own: the result is usable, but **tell the user**. `#reDepth` retries the AI; `#importDepth` / `#fDepth` accepts a provided map (light = near).
- Object selection (`#pickBtn`, `#cylDetect`) loads SlimSAM the first time; on failure: toast “La sélection IA a besoin d’internet la première fois · cadre estimé sans IA”.

## Checks before handing back

1. The logo used is **exactly** the provided file (not AVORA, not a redrawn version).
2. `#st2s` shows the depth source; if it is the quick estimate or an imported map, say so.
3. The exported file exists, opens, and has the expected size (the photo’s, unless the GPU limit applies).
4. Look at the export: logo visible, well placed, not cut by an unintended mask, no recolouring nobody asked for.
5. If the user needs to resume the work, also hand over the `.mockup.json`.
6. No console errors (apart from AI model loading warnings, if they led to the quick estimate).

## Limits and errors

| Message (toast `#toastT`) | Cause / what to do |
|---|---|
| “Ce navigateur ne prend pas en charge WebGL2” / “WebGL2 est requis par ce navigateur” | Enable WebGL2 (see § 2) |
| “Importe d’abord une photo” | Logo added before the photo |
| “Ce logo n’a pas pu être lu” | Invalid logo file: ask for it again, do not recreate it |
| “Le modèle IA n’a pas pu se charger · estimation rapide utilisée” | Network or Hugging Face unavailable |
| “La profondeur n’est pas encore prête” | `Fit to surface` before depth finished: wait |
| “Ce fichier n’est pas un projet valide” | The `.json` is not a Mockup project |
| “L’export a échoué : image trop grande pour ce navigateur” | Photo too large for the GPU |

Other limits: an SVG is rendered to an image (up to 4096 px per side); depth is computed on a working copy of at most 2048 px; changing the photo removes the logos and the history.

## Logic for every task

1. Ask for (or check you have) the user’s **photo** and **logo**. Without a logo, do not make one up.
2. Serve the folder over http, open `app.html` in a desktop window with WebGL2.
3. Import the photo, wait for depth, note its source.
4. Import the logo, place it, pick the material that fits the surface (Cylinder for bottle, can, mug).
5. Mask whatever must pass in front; keep blur at 0 unless asked.
6. Export the PNG (and the `.json` if useful), check (§ 9), then hand over the files saying what was set.
