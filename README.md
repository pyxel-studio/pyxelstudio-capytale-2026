# Pyxel Studio pour Capytale 2026

IDE en ligne pour [Pyxel](https://github.com/kitao/pyxel), intégré à la plateforme [Capytale](https://capytale.fr).

**Démo en ligne** : [https://pyxel-studio.github.io/pyxelstudio-capytale-2026/](https://pyxel-studio.github.io/pyxelstudio-capytale-2026/)

Pyxel Studio permet aux élèves et enseignants de créer des jeux et animations en Python avec la bibliothèque Pyxel, directement dans le navigateur.

## Fonctionnalités

- **Éditeur de code Python** (Ace Editor) avec coloration syntaxique
- **Éditeur de ressources Pyxel** (.pyxres) intégré pour le pixel art, les tilemaps, les sons et la musique
- **Exécution dans le navigateur** via Pyodide (Python WebAssembly) et Pyxel WASM
- **Gestion multi-fichiers** : .py, .pyxres, .pyxapp, .txt
- **Console** intégrée avec pause/play/clear
- **Import/export** de fichiers depuis/vers le disque local
- **Intégration Capytale** via le protocole de contrats `@capytale/app-agent`

## Architecture

Le projet est une application **100% statique** (HTML + CSS + JS), sans serveur backend ni étape de build.

```
index.html              Application principale (éditeur + gestion fichiers + RPC)
iframe-screen.html      Iframe d'exécution du code Python/Pyxel
iframe-pyxres.html      Iframe d'édition des ressources Pyxel (.pyxres)
assets/
  css/                  Bootstrap + styles personnalisés
  js/                   Ace Editor, Bootstrap, Split Grid
  fontawesome/          Icônes
  img/                  Logo, cheatsheet, palette
  wasm/                 Pyxel CSS
```

### Flux d'exécution

1. L'utilisateur écrit du code Python dans l'éditeur Ace (`index.html`)
2. Au clic sur **Play**, le code et les ressources sont sauvegardés dans `localStorage`
3. Une iframe (`iframe-screen.html`) est créée, charge Pyodide + Pyxel WASM
4. Le script Python lit les fichiers depuis `localStorage`, les écrit dans le VFS Pyodide, puis exécute le fichier cible

### Communication entre iframes

- **index.html <-> iframe-screen.html** : via `localStorage` (données partagées) et `window.pyxelContext` (accès au VFS Pyodide)
- **index.html <-> iframe-pyxres.html** : même principe, avec un mécanisme de sauvegarde silencieuse (F13 + lecture VFS) pour synchroniser les ressources éditées

### Stockage des données

Toutes les données sont stockées dans `localStorage` sous la clé `capytale_pyxel_studio_data` au format JSON :

```json
{
  "app.py": "import pyxel\n...",
  "res.pyxres": "UEsDBBQ...",
  "script2.py": "# mon script"
}
```

- Les fichiers `.py` et `.txt` sont stockés en texte brut
- Les fichiers `.pyxres` et `.pyxapp` sont stockés en base64

## Intégration Capytale

### Protocole

L'application utilise le package [`@capytale/app-agent`](https://www.npmjs.com/package/@capytale/app-agent) chargé dynamiquement via CDN (jsdelivr). Le contrat implémenté est **`simple-content(json):1`**.

### Contrat simple-content(json)

L'application expose au MetaPlayer :

| Méthode | Description |
|---------|-------------|
| `loadContent(content)` | Reçoit le contenu JSON depuis Capytale. `content` est un objet `{ "fichier": "contenu", ... }`. Si `null` ou vide, les fichiers par défaut sont chargés. |
| `getContent()` | Retourne l'objet JSON contenant tous les fichiers (code + ressources en base64). |
| `contentSaved()` | Callback appelé par Capytale après une sauvegarde réussie. |

L'application appelle côté MetaPlayer :

| Méthode | Description |
|---------|-------------|
| `contentChanged()` | Notifie Capytale qu'une modification a eu lieu (édition de code, ajout/suppression de fichier, modification de ressource). |

### Mode standalone

Quand l'application n'est pas dans une iframe Capytale (`window.parent === window`), le RPC est ignoré et l'application fonctionne de manière autonome avec `localStorage` uniquement.

### Code RPC

Le code d'initialisation RPC se trouve dans `index.html`, fonction `initRPC()` :

```javascript
const { getSocket } = await import("https://cdn.jsdelivr.net/npm/@capytale/app-agent/+esm");
const socket = getSocket();

socket.plug('simple-content(json):1', () => ({
    loadContent(content) { /* ... */ },
    getContent() { /* ... */ },
    contentSaved() { /* ... */ }
}));

socket.use('simple-content(json)', (sc) => {
    // sc.i.contentChanged() pour notifier les modifications
});

socket.plugsDone();
```

## Test dans le Capytale Playground

Le [Capytale Playground](https://github.com/capytale/capytale-playground) permet de tester l'application avant intégration dans Capytale.

### Configuration du Playground

| Paramètre | Valeur |
|-----------|--------|
| **IFrame URL** | `https://pyxel-studio.github.io/pyxelstudio-capytale-2026/` |
| **Contrat de sauvegarde** | `simple-content(json)` |
| **Permissions iframe** | `fullscreen` |

### Lancer en local

L'application étant 100% statique, il suffit de servir le dossier avec n'importe quel serveur HTTP :

```bash
# Avec Python
python -m http.server 8000

# Avec Node.js (npx)
npx serve .

# Avec PHP
php -S localhost:8000
```

Puis ouvrir `http://localhost:8000` dans le navigateur.

> **Note** : pour tester l'intégration Capytale, l'application doit être accessible en HTTPS (requis par le protocole postMessage/comlink).

## Dépendances externes (CDN)

| Dépendance | Usage | CDN |
|------------|-------|-----|
| [Pyxel WASM](https://github.com/kitao/pyxel) | Moteur de jeu rétro | `cdn.jsdelivr.net/gh/kitao/pyxel/wasm/` |
| [Pyodide](https://pyodide.org/) v0.23.1 | Runtime Python WebAssembly | `cdn.jsdelivr.net/pyodide/` |
| [@capytale/app-agent](https://www.npmjs.com/package/@capytale/app-agent) | Communication RPC avec Capytale | `cdn.jsdelivr.net/npm/@capytale/app-agent/` |
| [Ace Editor](https://ace.c9.io/) | Éditeur de code | Local (`assets/js/ace/`) |
| [Bootstrap](https://getbootstrap.com/) 5 | UI framework | Local (`assets/css/`, `assets/js/`) |
| [Font Awesome](https://fontawesome.com/) | Icônes | Local (`assets/fontawesome/`) |
| [Split.js](https://split.js.org/) | Panneaux redimensionnables | Local (`assets/js/split-grid.js`, `assets/js/split.min.js`) |

## Documentation Pyxel

- [Documentation officielle Pyxel](https://github.com/kitao/pyxel)
- [Guide utilisateur Pyxel](https://kitao.github.io/pyxel/web/user-guide/)
- [Pyxel Code Maker](https://github.com/kitao/pyxel/tree/main/web/code-maker)

## Documentation Capytale

- [Capytale Playground](https://github.com/capytale/capytale-playground)
- [MetaPlayer RPC](https://github.com/capytale/metaplayer-rpc)
- [Contrats Capytale](https://www.npmjs.com/package/@capytale/contracts)
