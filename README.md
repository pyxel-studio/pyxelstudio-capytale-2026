# Pyxel Studio v2

IDE en ligne pour [Pyxel](https://github.com/kitao/pyxel), integre a la plateforme [Capytale](https://capytale.fr).

Pyxel Studio permet aux eleves et enseignants de creer des jeux et animations en Python avec la bibliotheque Pyxel, directement dans le navigateur.

## Fonctionnalites

- **Editeur de code Python** (Ace Editor) avec coloration syntaxique
- **Editeur de ressources Pyxel** (.pyxres) integre pour le pixel art, les tilemaps, les sons et la musique
- **Execution dans le navigateur** via Pyodide (Python WebAssembly) et Pyxel WASM
- **Gestion multi-fichiers** : .py, .pyxres, .pyxapp, .txt
- **Console** integree avec pause/play/clear
- **Import/export** de fichiers depuis/vers le disque local
- **Integration Capytale** via le protocole de contrats `@capytale/app-agent`

## Architecture

Le projet est une application **100% statique** (HTML + CSS + JS), sans serveur backend ni etape de build.

```
index.html              Application principale (editeur + gestion fichiers + RPC)
iframe-screen.html      Iframe d'execution du code Python/Pyxel
iframe-pyxres.html      Iframe d'edition des ressources Pyxel (.pyxres)
assets/
  css/                  Bootstrap + styles personnalises
  js/                   Ace Editor, Bootstrap, Split Grid
  fontawesome/          Icones
  img/                  Logo, cheatsheet, palette
  wasm/                 Pyxel CSS
```

### Flux d'execution

1. L'utilisateur ecrit du code Python dans l'editeur Ace (`index.html`)
2. Au clic sur **Play**, le code et les ressources sont sauvegardes dans `localStorage`
3. Une iframe (`iframe-screen.html`) est creee, charge Pyodide + Pyxel WASM
4. Le script Python lit les fichiers depuis `localStorage`, les ecrit dans le VFS Pyodide, puis execute le fichier cible

### Communication entre iframes

- **index.html <-> iframe-screen.html** : via `localStorage` (donnees partagees) et `window.pyxelContext` (acces au VFS Pyodide)
- **index.html <-> iframe-pyxres.html** : meme principe, avec un mecanisme de sauvegarde silencieuse (F13 + lecture VFS) pour synchroniser les ressources editees

### Stockage des donnees

Toutes les donnees sont stockees dans `localStorage` sous la cle `capytale_pyxel_studio_data` au format JSON :

```json
{
  "app.py": "import pyxel\n...",
  "res.pyxres": "UEsDBBQ...",
  "script2.py": "# mon script"
}
```

- Les fichiers `.py` et `.txt` sont stockes en texte brut
- Les fichiers `.pyxres` et `.pyxapp` sont stockes en base64

## Integration Capytale

### Protocole

L'application utilise le package [`@capytale/app-agent`](https://www.npmjs.com/package/@capytale/app-agent) charge dynamiquement via CDN (jsdelivr). Le contrat implemente est **`simple-content(json):1`**.

### Contrat simple-content(json)

L'application expose au MetaPlayer :

| Methode | Description |
|---------|-------------|
| `loadContent(content)` | Recoit le contenu JSON depuis Capytale. `content` est un objet `{ "fichier": "contenu", ... }`. Si `null` ou vide, les fichiers par defaut sont charges. |
| `getContent()` | Retourne l'objet JSON contenant tous les fichiers (code + ressources en base64). |
| `contentSaved()` | Callback appele par Capytale apres une sauvegarde reussie. |

L'application appelle cote MetaPlayer :

| Methode | Description |
|---------|-------------|
| `contentChanged()` | Notifie Capytale qu'une modification a eu lieu (edition de code, ajout/suppression de fichier, modification de ressource). |

### Mode standalone

Quand l'application n'est pas dans une iframe Capytale (`window.parent === window`), le RPC est ignore et l'application fonctionne de maniere autonome avec `localStorage` uniquement.

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

Le [Capytale Playground](https://github.com/capytale/capytale-playground) permet de tester l'application avant integration dans Capytale.

### Configuration du Playground

| Parametre | Valeur |
|-----------|--------|
| **IFrame URL** | URL ou l'application est servie (ex: `https://capytale-pyxelstudio-v2/`) |
| **Contrat de sauvegarde** | `simple-content(json)` |
| **Permissions iframe** | `fullscreen` |

### Lancer en local

L'application etant 100% statique, il suffit de servir le dossier avec n'importe quel serveur HTTP :

```bash
# Avec Python
python -m http.server 8000

# Avec Node.js (npx)
npx serve .

# Avec PHP
php -S localhost:8000
```

Puis ouvrir `http://localhost:8000` dans le navigateur.

> **Note** : pour tester l'integration Capytale, l'application doit etre accessible en HTTPS (requis par le protocole postMessage/comlink).

## Dependances externes (CDN)

| Dependance | Usage | CDN |
|------------|-------|-----|
| [Pyxel WASM](https://github.com/kitao/pyxel) | Moteur de jeu retro | `cdn.jsdelivr.net/gh/kitao/pyxel/wasm/` |
| [Pyodide](https://pyodide.org/) v0.23.1 | Runtime Python WebAssembly | `cdn.jsdelivr.net/pyodide/` |
| [@capytale/app-agent](https://www.npmjs.com/package/@capytale/app-agent) | Communication RPC avec Capytale | `cdn.jsdelivr.net/npm/@capytale/app-agent/` |
| [Ace Editor](https://ace.c9.io/) | Editeur de code | Local (`assets/js/ace/`) |
| [Bootstrap](https://getbootstrap.com/) 5 | UI framework | Local (`assets/css/`, `assets/js/`) |
| [Font Awesome](https://fontawesome.com/) | Icones | Local (`assets/fontawesome/`) |
| [Split.js](https://split.js.org/) | Panneaux redimensionnables | Local (`assets/js/split-grid.js`, `assets/js/split.min.js`) |

## Documentation Pyxel

- [Documentation officielle Pyxel](https://github.com/kitao/pyxel)
- [Guide utilisateur Pyxel](https://kitao.github.io/pyxel/web/user-guide/)
- [Pyxel Code Maker](https://github.com/kitao/pyxel/tree/main/web/code-maker)

## Documentation Capytale

- [Capytale Playground](https://github.com/capytale/capytale-playground)
- [MetaPlayer RPC](https://github.com/capytale/metaplayer-rpc)
- [Contrats Capytale](https://www.npmjs.com/package/@capytale/contracts)
