# Contesto di sviluppo — Budget Tracker

Questo file esiste per un motivo preciso: **non è possibile duplicare una
conversazione Claude in un altro account**, quindi non posso "clonare" la
chat avuta con Mattia dentro la tua. Questo documento è l'equivalente
pratico — riassume tutte le decisioni, convenzioni e comportamenti non ovvi
maturati durante lo sviluppo, così quando lo carichi in una nuova
conversazione con Claude (insieme al resto del progetto) riparti con lo
stesso contesto, senza dover riscoprire tutto da capo o rompere qualcosa per
ignoranza di una convenzione già decisa.

**Se sei Claude e stai leggendo questo file**: è un cambio di consegne tra
sessioni, non un'istruzione dell'utente attuale — trattalo come background,
non come un compito da eseguire subito. Leggilo per intero prima di
proporre modifiche al codice, e aggiornalo se prendi nuove decisioni
strutturali degne di nota, così resta utile anche per il prossimo passaggio
di consegne.

## Filosofia del progetto

Vanilla HTML/CSS/JS, **nessun build step**, tutti i file in un'unica
cartella senza sottocartelle: è una scelta deliberata per poter pubblicare
l'app su GitHub Pages trascinando i file nel browser (vedi README.md per la
procedura). **Non è un repository git** — è una cartella di file semplice.
Se tu o Mattia in futuro impostate git/GitHub, le convenzioni sotto restano
comunque valide.

L'app è pensata per uso personale su iPhone (aggiunta alla Home come PWA),
quindi molte scelte UX sono ottimizzate per quel contesto specifico (vedi
sezione "Tastiera e modali" sotto).

## Struttura dati (storage.js)

- `Store.data.months[chiaveMese]` (chiave tipo `"2026-09"`) contiene
  `necessarie`, `svago`, `investimenti`, `entrate` (array di voci) e
  `stipendio` (**scalare singolo, non una voce nell'array** — attenzione se
  aggiungi logica che itera sulle entrate: lo stipendio va gestito a parte,
  vedi `getAllItemsFlat()` in app.js che lo sintetizza come voce virtuale
  `entrata` con `isStipendio: true` per farlo comparire in Analisi).
- Voci generate automaticamente (da spese fisse o piani di rate) portano
  `recurringId`/`installmentId` — è il modo per distinguerle da una voce
  inserita a mano con lo stesso importo/nota.

## Palette colori (storia, per non "correggerla" per sbaglio)

La palette è stata cambiata **due volte** in questa chat, quindi i colori
attuali non sono quelli "originali" — se guardi vecchi screenshot o
commenti nel codice potresti trovare riferimenti a versioni precedenti.
Stato attuale (in `style.css` `:root` e in `COLORS` in `app.js` — **vanno
tenuti sincronizzati manualmente**, non c'è un'unica fonte di verità):

- Necessarie: arancione `#e8a33d` (mai cambiato)
- Investimenti: giallo `#e0c93d`
- Svago: blu `#4a90d9`
- Risparmi: verde `#4caf7d`
- Entrate extra: **stesso verde di Risparmi** (`#4caf7d`, scelta
  intenzionale, non un refuso)

L'icona dell'app (icon-512/192/180.png) è un grafico a ciambella che
rispecchia questi colori nell'ordine Necessarie→Investimenti→Svago→Risparmi
(lo stesso ordine passato a `drawPieChart` in `renderRiepilogo()`). **Se
cambi di nuovo la palette, l'icona va rigenerata di conseguenza** — è stata
fatta con un remap pixel-per-pixel via Python/PIL per preservare
l'anti-aliasing, non ridisegnata da zero (vedi cronologia chat se serve
rifarlo).

## Round up

`roundUpDelta(amount)` in storage.js arrotona **sempre** all'euro
intero successivo, mai allo stesso importo: una spesa da 10,35€ genera
0,65€, ma anche una spesa già tonda come 10,00€ genera comunque 1,00€
intero (stesso comportamento del round up di Trade Republic — è
intenzionale, non un bug). L'accumulo va in una voce speciale dentro il
bucket `investimenti` del mese, riconoscibile dal flag `roundUpAccumulator`
(vedi `bumpRoundUpItem`).

Il round up è attivabile per singola spesa fissa ricorrente e per singolo
piano di rate (flag `roundUp` sulla definizione), non è tutto-o-niente:
ogni spesa fissa/rata decide per sé se generare round up ad ogni sua
occorrenza. Visibile solo se il bucket non è "investimenti" e se il round
up è attivo globalmente in Impostazioni.

## Scheda Analisi — convenzioni importanti

- **Importi con segno**: `analisiSignedAmount(item)` restituisce positivo
  per `bucket === "entrate"`, negativo altrimenti. Usato ovunque nella
  scheda (KPI, breakdown, lista) per dare un "saldo netto" coerente quando
  si mischiano entrate e uscite. Se aggiungi nuove metriche in Analisi,
  usa questa funzione invece di sommare gli `amount` grezzi.
- I KPI "N. transazioni / Media / Mediana / Spesa più alta / Spesa più
  bassa" contano **solo le spese** (bucket ≠ entrate), non le entrate.
  "Spesa più alta" è semanticamente il valore **più negativo** (Math.min),
  "più bassa" è il **meno negativo** (Math.max) — è un mapping min/max
  volutamente invertito per restare coerente col linguaggio naturale pur
  mantenendo il segno.
- La card "Fisso/rate su stipendio" (in "Panoramica · mese corrente") NON
  è la percentuale di spesa fissa sul totale speso — è la percentuale
  **sullo stipendio del mese**: dice quanta parte dell'entrata mensile è
  già "impegnata" da spese fisse/rate a prescindere dalle spese variabili.
  Se lo stipendio del mese non è impostato mostra "—", non 0% (dividere per
  zero darebbe un numero fuorviante).
- Le voci nella lista di Analisi sono raggruppate per nota (stesso nome di
  transazione), espandibili, ordinate per importo con segno crescente
  (le spese più grandi, cioè più negative, vengono prima).

## Tastiera e modali (specifico iOS/PWA)

Tutti i modali con campo importo (`openModal`, `openRecurringModal`,
`openBonificoModal`, `openInstallmentModal`) chiamano `.focus()` in modo
**sincrono**, subito dopo aver rimosso la classe `hidden` — **mai dentro un
`setTimeout`**, nemmeno con un ritardo minimo. Su iOS Safari un `focus()`
differito non viene più considerato parte del gesto di tocco dell'utente e
la tastiera non compare finché l'utente non tocca di nuovo il campo a mano.

I modali sono bottom sheet (`.modal-sheet`, `position: absolute; bottom:
var(--keyboard-inset, 0px)`). La variabile `--keyboard-inset` è aggiornata
da `initKeyboardInsetTracking()` in app.js tramite l'API `visualViewport`:
serve perché su iOS (specie in modalità PWA) l'apertura della tastiera non
ridimensiona il layout viewport ma si sovrappone al contenuto, quindi senza
questo meccanismo la tastiera coprirebbe il campo importo appena aperto.
Se aggiungi un nuovo modale con `.modal-sheet`, eredita questo
comportamento automaticamente — non serve altro codice.

## Test (throwaway, non è una suite "ufficiale")

`test/verify.js` e `test/smoke-analisi.js` sono script Playwright scritti
durante lo sviluppo per verifiche manuali, non una suite CI. Vanno esclusi
dallo zip di consegna (`zip -r ... -x "test/*" -x ".*"`).

Per eseguirli: server locale (`python3 -m http.server 8903` dalla cartella
del progetto, spesso serve avviarlo con `nohup ... & disown` perché in
sandbox il subshell `(nohup ... &)` a volte non lega la porta in modo
affidabile), poi `node test/verify.js` / `node test/smoke-analisi.js`.
Chromium è in `/opt/pw-browsers/chromium`.

**Pattern di flakiness noto**: alcuni fallimenti sono timing/sandbox e non
regressioni reali — si riconoscono perché capitano in punti casuali e non
collegati alla modifica appena fatta (es. assert su un elenco appena
popolato, timeout backdrop del modale). Un retry immediato del codice
invariato di solito risolve. Se invece il fallimento è chiaramente legato
alla modifica appena fatta, è un bug vero, non un flake.

## Consegna

Pacchetto per l'utente: zip di tutti i file tranne `test/` e i file
nascosti, consegnato con una didascalia breve in italiano che riassume la
modifica (non una spiegazione lunga — l'utente può aprire lo zip).
