# Stefano Mondino - Hugo Site

Sito personale costruito con Hugo e il tema [Mana](https://github.com/Livour/hugo-mana-theme).

## Stack

- **Hugo** v0.157+ (Extended) - gestito tramite `mise`
- **Node** 22 - gestito tramite `mise`
- **Tema**: Mana (submodule git in `themes/mana`)

## Comandi

- `mise exec -- hugo server -D` — avvia il server di sviluppo (con draft)
- `mise exec -- hugo` — build del sito in `public/`
- `mise install` — installa le dipendenze (hugo + node) tramite mise

## Struttura del progetto

```
hugo.toml              # Configurazione principale del sito
mise.toml              # Gestione versioni strumenti (hugo, node)
content/
  posts/               # Articoli del blog (markdown)
  about/index.md       # Pagina About
static/                # File statici (favicon, immagini non processate)
assets/images/         # Immagini processate da Hugo (resize, ottimizzazione)
themes/mana/           # Tema Mana (submodule git - NON modificare direttamente)
```

## Convenzioni contenuti

- I post vanno in `content/posts/` come file `.md`
- Front matter obbligatorio: `title`, `date`, `draft`, `tags`
- La pagina About usa layout `about` ed e' in `content/about/index.md`
- Lingua predefinita: italiano (`it`)

## Tema Mana - Note

- Il tema e' un git submodule: non modificare file dentro `themes/mana/`
- Per personalizzare layout, creare override in `layouts/` alla root del progetto
- Per personalizzare CSS, creare file in `assets/css/` alla root
- Supporta admonitions stile GitHub/Obsidian (NOTE, TIP, WARNING, ecc.)
- Supporta dark/light mode toggle
- La ricerca full-text richiede output JSON (gia' configurato in `hugo.toml`)

## Configurazione

- `hugo.toml` contiene tutta la configurazione: menu, params, social links
- Single language (italiano), single content folder
- Syntax highlighting con temi Catppuccin

## Regole per Claude

- Usa sempre `mise exec -- hugo` per eseguire comandi hugo
- Non modificare mai file dentro `themes/mana/`
- Per override di template, copia il file da `themes/mana/layouts/` a `layouts/` mantenendo lo stesso path
- I contenuti sono in italiano salvo diversa indicazione
- Dopo modifiche alla configurazione o ai layout, verifica con `mise exec -- hugo` che il build funzioni
