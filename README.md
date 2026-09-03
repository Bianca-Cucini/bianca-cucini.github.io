# biancacucini.github.io

Sito personale di Bianca Cucini, violista da gamba (formatasi alla Schola Cantorum
Basiliensis di Basilea con Paolo Pandolfo). Pubblico target internazionale
(IT / EN / DE).

Nessun framework, nessun build step: HTML/CSS/JS scritti a mano, ospitati su
**GitHub Pages** con dominio personalizzato `biancacucini.com` (vedi `CNAME`).
Per pubblicare una modifica basta fare commit + push su `main`.

## Struttura delle pagine

Il sito **non** usa sottocartelle per lingua (`/en/`, `/de/`) ma file separati
con suffisso, tutti nella root:

```
index.html                        landing page — solo selettore di lingua (IT/EN/DE)
home-it.html / home-en.html / home-de.html
agenda-it.html / agenda-en.html / agenda-de.html
collaborazioni-it.html / -en.html / -de.html
contatti-it.html / -en.html / -de.html
media-it.html / -en.html / -de.html
quinton-it.html / -en.html / -de.html
```

**Importante**: `index.html` è l'unica pagina con un selettore di lingua vero
e proprio (tre pulsanti che portano a `home-it/en/de.html`). Le pagine interne
**non hanno un cambio-lingua in pagina** — la lingua giusta viene raggiunta
tramite i link di navigazione interni (coerenti per lingua) o tramite i tag
`hreflang` che aiutano Google a proporre la versione giusta nei risultati di
ricerca. Se in futuro si vuole aggiungere un vero cambio-lingua in pagina, va
costruito da zero.

Ogni pagina ha in `<head>`: `<link rel="canonical">` e 4 `<link rel="alternate"
hreflang="...">` (it, en, de, x-default) che puntano alle rispettive versioni.
Per il gruppo home, `x-default` punta a `index.html`; per tutte le altre pagine
punta alla versione inglese (lingua di ripiego per pubblico internazionale).

## Stile — palette e font

Font (Google Fonts, caricati via `<link>` in ogni pagina):
- **Josefin Sans** (300–600) — titoli, H1/H2, elementi identitari
- **Work Sans** (300–500/600) — testo corrente

**Attenzione — esistono due palette leggermente diverse**, non un unico set di
variabili CSS condiviso. Chi continua il lavoro dovrebbe saperlo prima di
"correggere" quella che sembra un'incoerenza:

- **Famiglia "calda"** (home, agenda, collaborazioni, contatti):
  `--ink:#303131` `--paper:#F3EEE4` `--paper-soft:#EEEBE4` `--stone:#D0C9BB`
  `--accent:#294A5B` `--accent-dark:#1c333f` `--cognac:#995C3F` `--slate:#737978`
- **Famiglia "azzurra"** (index.html, media, quinton):
  `--ink:#17191b` `--paper:#f6f2e8` `--paper-soft:#f0ebdf` `--mist:#dde6e2`
  `--azure:#4f83a0` `--azure-deep:#2c5468` `--slate:#9a9d9f`

Ogni pagina definisce le sue variabili in un blocco `:root` dentro un
`<style>` inline in testa al file (nessun CSS esterno condiviso).

Librerie esterne usate: Bootstrap 4.6.2 (grid/utility), jQuery slim 3.7.1 +
Popper.js (dipendenze Bootstrap), Font Awesome 5.15.4 (icone social e icone
varie), Leaflet 1.9.4 (solo nella pagina Agenda, per le mappe).

**Nota**: esistono due implementazioni di gallery fotografica diverse e
indipendenti — `quinton-it/en/de.html` usa una modale Bootstrap
(`openGallery`/`#galleryModal`), mentre `media-it/en/de.html` usa una
lightbox scritta da zero (`openLightbox`/`lbNav`). Non è codice condiviso:
una modifica in un file non si riflette sull'altro.

## Pagina Agenda — come funziona

Questa è la pagina con più logica del sito, pensata per permettere a Bianca di
aggiornare il calendario concerti **senza toccare mai il codice**, tramite un
foglio Google.

### Flusso dei dati

```
Foglio Google "Concerti"  (Bianca modifica qui, come sempre)
        │
        │  1 volta al giorno (trigger a tempo)
        ▼
Google Apps Script (agenda-sync)  →  legge il foglio, pubblica via GitHub API
        │
        ▼
agenda-data.json  (nella root del repo, stesso dominio del sito)
        │
        │  fetch() dal browser del visitatore
        ▼
agenda-it/en/de.html  →  fetch('agenda-data.json') → initAgenda(dati)
        │
        │  se il fetch fallisce (file irraggiungibile)
        ▼
FALLBACK_CONCERTS  (array statico dentro l'HTML, 14 concerti reali)
```

**Perché questa architettura**: in origine il sito leggeva Google Sheets
direttamente dal browser del visitatore (`fetch` verso `docs.google.com`).
Le reti aziendali che bloccano Google Docs/Sheets per policy impedivano il
caricamento dell'agenda a chi visitava da lì. Spostando la pubblicazione dei
dati su un file dello stesso dominio (`agenda-data.json`), il sito non
contatta più Google in nessun momento: la richiesta è identica a quella per
qualunque altra immagine o script del sito, quindi non è bloccabile per
categoria da un firewall aziendale.

### I file coinvolti

- **`agenda-data.json`** (root del repo): snapshot dei dati del foglio,
  in formato array di oggetti con questi campi esatti:
  `date, date_end, time, venue, city, festival, ensemble, program, lat, lng, info`.
  Aggiornato automaticamente — non va modificato a mano.
- **`apps-script/agenda-sync.gs`**: copia di riferimento/documentazione dello
  script. **Non viene eseguito dal sito**: va incollato manualmente
  nell'editor Google Apps Script agganciato al foglio Google (Estensioni →
  Apps Script), come file *separato* accanto allo script di geocoding già
  presente lì (vedi sotto). Le istruzioni di installazione sono nei commenti
  in testa al file stesso.
- **`agenda-it/en/de.html`**: contengono la funzione `initAgenda(concerts)`
  che riceve l'array di concerti e disegna: la timeline verticale dei
  prossimi concerti, l'archivio raggruppato per anno (collassabile), e due
  mappe Leaflet (prossimi/archivio). La distinzione "prossimo vs passato" è
  **calcolata dinamicamente** confrontando `date`/`date_end` con la data
  odierna (funzione `isUpcoming`) — non è un campo da impostare a mano.

### Il foglio Google

- Il tab con i dati si chiama **"Concerti"**.
- Sullo stesso foglio Google gira **un secondo script, indipendente**, già
  presente prima di questo lavoro: calcola in automatico le coordinate
  lat/lng quando si modificano le colonne venue/città (trigger `onEdit`).
  I due script convivono nello stesso progetto Apps Script come file
  separati e non hanno conflitti di nomi.
- Pubblicazione CSV del foglio (usata storicamente, ora non più letta
  direttamente dal sito, ma utile come riferimento/backup):
  `https://docs.google.com/spreadsheets/d/e/2PACX-1vQwq0Sgn_ptcKAm437zvsWvl6MWd1nIruGdLtm2Tkc3llKFXy4arHmOAYwwo-PjjUtqSvPdeb5jVVRg/pub?gid=0&single=true&output=csv`

### Autenticazione verso GitHub

Lo script Apps Script scrive su GitHub tramite un **Personal Access Token
fine-grained**, salvato come proprietà script (`GITHUB_TOKEN`), con permesso
limitato a `Contents: Read and write` sul solo repo `bianca-cucini.github.io`.

**Da ricordare**: GitHub impone una scadenza massima di **366 giorni** ai
token fine-grained sui repository di organizzazioni (non è configurabile,
non esiste "senza scadenza" in questo caso) — quindi va **rigenerato una
volta all'anno**. Per non doversene accorgere solo quando l'agenda smette di
aggiornarsi, lo script invia automaticamente un'**email di avviso** in caso
di fallimento (token scaduto o altro errore), all'indirizzo Google che
possiede lo script Apps Script (o a `NOTIFY_EMAIL` se impostata come
proprietà script). Se il token scade e non viene rinnovato, il sito non si
rompe: continua semplicemente a mostrare l'ultimo `agenda-data.json`
sincronizzato con successo.

### Il fallback statico

`FALLBACK_CONCERTS`, dentro ciascun `agenda-*.html`, è l'ultima rete di
sicurezza se anche `agenda-data.json` fosse irraggiungibile. Contiene 14
concerti reali (verificati) come esempio, **non** l'intero storico — per
quello c'è `agenda-data.json`. Se lo si modifica, va rispettato lo stesso
schema di campi elencato sopra (in passato conteneva concerti sbagliati,
appartenenti a un'altra musicista il cui template era stato riadattato per
questo sito: episodio ormai risolto, ma un promemoria a controllare sempre
che i dati statici corrispondano davvero a Bianca).

## Press kit

Pagina Media (`media-*.html`) offre due download diretti e statici, senza
alcuna logica dietro:
- `press_kit/Dossier_BiancaCucini_2026.pdf` (~2,2 MB)
- `press_kit/Photos_BiancaCucini_2026.zip` (~8,6 MB)

Per aggiornarli, va sostituito il file nella cartella `press_kit/` mantenendo
lo stesso nome (o aggiornando anche i link `href` nelle tre lingue se il nome
cambia, es. per l'anno).

## SEO

- `sitemap.xml` e `robots.txt` in root, con tutte le 19 pagine indicizzabili.
- Meta description uniche e scritte a mano per ogni pagina/lingua (evitare di
  lasciarne di generiche o duplicate tra lingue diverse: Google le tronca
  oltre ~155-160 caratteri).
- Favicon dichiarata anche a 180×180px (oltre a 16/32px) — Google richiede
  almeno 48×48px per mostrarla nei risultati di ricerca; l'aggiornamento in
  SERP può richiedere giorni/settimane anche dopo il deploy.

## Cose particolari da ricordare

- Il repository è **pubblico** (rilevante se in futuro si valutano GitHub
  Actions: sui repo pubblici sono gratuite e senza limiti di minuti).
- `index.html` e le tre `home-*.html` sono pagine diverse: `index.html` è
  solo lo splash/selettore lingua con foto a schermo intero, `home-*.html` è
  la vera homepage con i contenuti.
- Non esiste un file CSS/JS condiviso: ogni pagina è autonoma, con `<style>`
  e `<script>` inline propri. Una modifica di stile "globale" va ripetuta
  manualmente su tutte le pagine/lingue interessate.

---

© 2026 Bianca Cucini — All rights reserved
