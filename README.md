# GUGGIAROM

Pagina singola: la scritta **GUGGIAROM** come oggetto 3D vero renderizzato in ASCII,
su un monitor CRT, più una sezione contatti minimale.
Un file, zero dipendenze, nessuna transform CSS.

## Come si guarda

```
open index.html
```

Muovi il mouse: la scritta **ti segue con lo sguardo**, come una testa che ti tiene
d'occhio nella stanza. Se nessuno tocca il mouse, lo sguardo vaga piano da solo.
Sotto la hero (alta `100vh`) ci sono le schede delle persone e i contatti; in alto
c'è un menu fisso che ci porta.

## Menu e invito a scorrere

Lo **scorrimento non è animato a mano**: è `scroll-behavior:smooth` sull'`html`, una
riga di CSS, e i link del menu ci passano dentro da soli. Con
`prefers-reduced-motion` si salta a destinazione senza scorrere. Le sezioni hanno
`scroll-margin-top:76px`, altrimenti si fermerebbero dietro al menu fisso.

L'invito in fondo alla hero è una scritta `scorri` e un filo verticale lungo cui
**scende una scintilla**. Dice "giù" molto meglio di una freccia ferma: è il
movimento a indicare la direzione, non il simbolo. Alla prima rotella (10% di
`innerHeight`) sparisce — ha fatto il suo lavoro.

Il **marchio** in alto a sinistra compare solo dopo il 72% della hero: finché la
scritta gigante è sullo schermo, ripeterlo in alto sarebbe dirlo due volte. Le due
soglie sono diverse apposta — l'invito sparisce alla prima intenzione di scorrere,
il marchio aspetta di servire davvero.

### L'accompagnamento dalla hero

Chi arriva vede una schermata intera occupata da una scritta. Se scorre e si ferma
a metà strada, la pagina lo accompagna giù fino alle schede con una discesa di
1100ms su una curva che parte piano, prende velocità in mezzo e si posa piano.

**Come non farlo: intercettare la rotella e annullarla.** Ci ho provato, in due
versioni, e non funziona per un motivo strutturale: una volta che il browser ha
iniziato a scorrere, i `wheel` successivi diventano **non annullabili**
(`cancelable === false`) e `preventDefault` fallisce in silenzio. Su trackpad
l'inerzia va avanti da sola per circa un secondo e passa sopra a qualunque
animazione le si metta contro — era lei a trascinare la pagina fino ai contatti,
non il codice. Anche la finestra di silenzio per distinguere il gesto dalla sua
coda era una pezza sopra al problema sbagliato.

**Come farlo: aspettare.** Non si tocca lo scorrimento dell'utente e non si annulla
niente. Si guarda quando la pagina si è **fermata** — 150ms senza un solo evento di
scroll, cioè a inerzia esaurita — e solo allora, se è rimasta a mezz'aria dentro la
hero e stava andando in giù, la si accompagna il resto della strada.

Ne vengono gratis tutte le proprietà che prima si dovevano difendere a mano:

- **non può incatenarsi**: il bersaglio è uno solo e parte solo dalla hero;
- **non combatte l'inerzia**: quando agisce, l'inerzia è già finita per definizione;
- **niente listener non-passive**, quindi nessun costo su ogni scorrimento della pagina;
- qualunque gesto durante la discesa la ferma — non è un binario;
- si riarma solo tornando in cima, e non parte se si sta scorrendo all'indietro.

### Dove atterra una sezione

Non al suo bordo. Lì ci sono la riga divisoria e 7vh di respiro, quindi arrivandoci
si vedeva un filo, del vuoto, e il titolo ancora in basso: la discesa sembrava
essersi fermata a metà. Il punto d'arrivo giusto è il **titolo**, a 96px dal bordo
alto — appena sotto al menu, con il contenuto che comincia subito dopo.

`landing(sezione)` restituisce quel punto, e la usano sia la discesa automatica sia
i **link del menu**: se no il menu e lo scroll automatico porterebbero a due punti
diversi della stessa sezione. Per i link lo scorrimento resta quello del browser —
`scroll-behavior:smooth` è già attivo sull'`html`, quindi la `scrollTo` è morbida di
suo.

L'unico dettaglio che resta è che `scroll-behavior:smooth` va spento per la durata
della discesa automatica: con quella proprietà attiva ogni `scrollTo` programmatica diventa a
sua volta smooth, e il browser inseguirebbe la nostra posizione di un frame fa
mentre noi ne scriviamo una nuova.

Con `prefers-reduced-motion` non si attiva.

### Si atterrava nel vuoto

`.contact` aveva `padding-top:14vh` e la riga divisoria altri `9vh` sotto: 23vh di
niente fra il bordo della sezione e il titolo. Cliccando "contatti" nel menu si
arrivava su uno schermo vuoto, con `CONTATTI` ancora sotto la piega — e la stessa
cosa succedeva scendendo. Ora le due sezioni si aprono con lo stesso ritmo (`6vh`
di padding, `7vh` sotto la riga) e il titolo atterra subito sotto al menu.

La voce accesa del menu è quella della sezione **a metà schermo**, non quella che
entra per prima: un `IntersectionObserver` con `rootMargin: -48% 0px -48% 0px`
riduce la zona utile a una fascia sottile al centro, così non ce n'è mai due accese
insieme e il passaggio avviene dove l'occhio sta già guardando.

Il menu non ha una barra opaca, solo un velo che sfuma verso il basso, così il
monitor resta una superficie sola invece di essere tagliato in due.

### Due bug che valeva la pena capire

**Il menu si vedeva ma a volte non si cliccava.** Stava a `z-index:1`, cioè lo
stesso livello di `.portrait` e `.contact`. A pari livello decide l'ordine nel DOM,
e le sezioni vengono dopo: scorrendo, la sezione passava *sopra* al menu fisso e si
prendeva lei i click. Ora lo stack è esplicito — contenuto `1`, menu `2`, vetro CRT
`3` — e il menu resta comunque dentro al monitor, sotto al vetro.

**Il testo dell'interfaccia era nero su nero.** Menu, kicker, invito a scorrere e
footer usavano `--dark`, che però è il colore dell'**ombra del solido 3D**: è nato
per essere quasi nero, perché lì serve a far sprofondare una faccia. Su testo di
10px dava un contrasto di circa 2.6:1, illeggibile. Sono due mestieri diversi e ora
sono due variabili diverse: `--dark` resta la parete in ombra del render, `--dim` è
il tono spento dell'interfaccia.

Alzare la variabile però non bastava, e capire perché è la parte interessante: **il
contrasto nominale non è quello che si vede.** Il testo dell'interfaccia va letto
*attraverso* tre strati che lo scuriscono — le scanline in `multiply`, la maschera a
fosfori, e soprattutto la vignettatura, che è massima proprio ai bordi, cioè dove
sta il menu. Un `#59b3aa` che su nero misura 7:1 arriva all'occhio molto più
spento. È servito lavorare su tre fronti:

| | |
|---|---|
| `--dim` a `#8fe4d8` | il tono di partenza, molto più su |
| un alone di fosforo sul testo | non è decorazione: allarga il tratto di mezzo pixel, e così la lettera sopravvive alle scanline, che a 10px cancellano una riga di glifo su due |
| vignettatura alleggerita | `inset 0 0 9vh` a `.7` invece di `14vh` a `.85`, e le bande delle scanline da `.34` a `.28`. La fascia scura arrivava fin dentro al menu e se lo mangiava; la curvatura del tubo si legge lo stesso con meno |

Il menu è anche salito da 10.5px a 11.5px con meno spaziatura: a quella misura le
scanline ne mangiavano troppo.

## Come funziona

Tutto in `index.html`. Non è un'estrusione finta: è un **raycaster**.

1. **Il solido** — la parola è un prisma. La sezione è la bitmap delle lettere
   (`FONT`, 7 righe, `#` = pieno), estrusa lungo `z` per `DEPTH` unità.
   È centrato nell'origine, quindi ruotando non si sposta: gira su sé stesso.
2. **Il raggio** — per ogni cella di testo si costruisce un raggio dalla camera
   (prospettica, distanza `CAM`) verso quel punto dello schermo. Il raggio viene
   portato in spazio oggetto con la **trasposta** della matrice di rotazione — vale
   sia per la direzione sia per l'origine.
3. **L'intersezione** — test analitico contro l'AABB del prisma per trovare
   l'intervallo utile, poi marcia a passo `STEP` campionando la bitmap. Il primo
   punto pieno è la superficie. L'occlusione viene gratis: si ferma al primo colpo.
4. **La normale** — se il raggio è entrato dalla faccia `z` e colpisce subito, è un
   *tappo*: normale `(0,0,±1)`. Altrimenti è una *parete laterale* e la normale si
   ricava dal gradiente 2D della bitmap attorno al punto.
5. **Shading piatto** — niente rampa di luminosità: una rampa a molti livelli su
   lettere larghe pochi caratteri diventa puntinato illeggibile (provato, scartato).
   Ogni faccia ha il suo blocco e il suo colore: `█` faccia frontale, `▓` parete
   laterale in luce, `▒` parete in ombra, `░` faccia posteriore. I caratteri
   consecutivi della stessa classe vengono accorpati in un solo `<span>`.

Il risultato: ruotando, la faccia frontale si scorcia e si deforma, e sui lati
compaiono le pareti del solido — restando leggibile.
compaiono le pareti dell'estrusione. Cambiano solo caratteri e colore.

## Le intestazioni di sezione

`CHI SIAMO` e `CONTATTI` non sono testo in maiuscoletto: sono **ASCII vera**,
montata dallo stesso font a caratteri della hero. Selezionandole e incollandole
altrove si ottiene il banner, non una parola — ed è il punto: se la scritta grande
è fatta di caratteri, non ha senso che i titoli siano tipografia normale.

```
 ████   ████  █    █ ██████  ████  ██████ ██████ ████
█    █ █    █ ██   █   ██   █    █   ██     ██    ██
█      █    █ █ █  █   ██   █    █   ██     ██    ██
█      █    █ █  █ █   ██   ██████   ██     ██    ██
█      █    █ █   ██   ██   █    █   ██     ██    ██
█    █ █    █ █    █   ██   █    █   ██     ██    ██
 ████   ████  █    █   ██   █    █   ██     ██   ████
```

Il carattere pieno è `█`, la stessa faccia frontale del solido 3D.

### Posizione e allineamento

Il banner è **centrato**, e la cosa è detta sul banner stesso invece di essere
ereditata dal contenitore. Prima `CONTATTI` si centrava perché `.contact` ha
`text-align:center`, e `CHI SIAMO` restava a sinistra perché `.portrait` no: due
intestazioni identiche allineate in modo diverso per un dettaglio che non le
riguarda.

Sopra c'è ora un `.rule`, lo stesso filo che stacca i contatti dalla sezione
precedente. Era scoped a `.contact`; senza, `.portrait` si attaccava alla hero
senza che niente dicesse che lì comincia un'altra cosa. Nella sezione delle schede
è vincolata a 940px, la larghezza della colonna del contenuto — un filo più largo
delle schede si noterebbe.

Sotto, il margine è passato da 30px a `clamp(46px, 7vh, 84px)`: a 30px il titolo
stava più vicino alla scheda di Andrea che al filo sopra, e si leggeva come
l'etichetta di quella scheda invece che come l'intestazione della sezione.

### Il corpo è uno solo per tutti i banner

Prima ognuno si dimensionava **sul proprio contenitore** — `.portrait` è larga
940px, `.contact` 760 — e siccome `CHI SIAMO` è 55 colonne e `CONTATTI` 53, due
intestazioni che devono leggersi come la stessa cosa venivano di due misure
diverse. Un titolo ha la sua taglia; è il contenitore a doverla rispettare, non il
contrario.

Ora si parte da un corpo fisso (`MAX = 12px`) e lo si abbassa solo se il banner più
largo non ci sta. Non si riempie la larghezza disponibile — riempirla era l'altro
motivo per cui su schermo grande diventavano enormi. A 12px `CHI SIAMO` viene
396×84px: si legge come un titolo e resta chiaramente sotto alla scritta della
hero.

### Il font è uno solo

`ASCII_FONT` sta fuori da entrambi gli script e non appartiene a nessuno dei due,
perché serve a due cose molto diverse: la hero lo usa come **sezione di un solido
da estrudere**, le intestazioni come **disegno piatto**. Le sette lettere di
GUGGIAROM sono rimaste identiche al carattere — la hero deve renderizzare
esattamente com'era — e le altre diciannove seguono la stessa regola: 6 colonne,
aste spesse 1, tranne la `I` che ne vuole 4 o resterebbe un lastrone.

Aggiungere un'intestazione ora è un `<pre class="banner" data-word="…">` e basta.

## Parametri (in cima allo script)

| | |
|---|---|
| `WORD` / `GAP` | parola e spaziatura tra le lettere |
| `DEPTH` | spessore del solido |
| `MAX_YAW` / `MAX_PITCH` | escursione della rotazione, in radianti |
| `CAM` (in `layout()`) | distanza camera; più grande = prospettiva più morbida |
| `STEP` | passo di marcia: più piccolo = più preciso e più lento |
| `CH_FACE` / `CH_LIT` / `CH_DARK` / `CH_BACK` | i quattro blocchi |
| `LX/LY/LZ` | direzione della luce |

Colori del render: le variabili CSS `--face`, `--lit`, `--dark`, `--back`. Il testo
dell'interfaccia ha la sua, `--dim`: vedi sotto, il perché non è ovvio.

## Trattamento CRT

Sopra al render c'è uno strato di atmosfera retro-futurista, tutto in CSS, che non
tocca la geometria:

- **Fosfori ciano** — palette fredda, con la faccia frontale come punto più caldo.
- **Aberrazione cromatica + bloom** — `text-shadow` sul `<pre>`: una copia magenta
  spostata a sinistra, una ciano a destra, più un alone diffuso.
- **Scanline** (`.crt.lines`) e **maschera a fosfori** verticale (`.crt.mask`).
- **Vignettatura** (`.crt.glass`) per suggerire la curvatura del tubo.
- **Flicker** impercettibile sull'opacità del render.

Tutti gli strati sono `pointer-events:none`, quindi il mouse continua a raggiungere
la pagina. Con `prefers-reduced-motion` flicker e sweep si fermano.

## Struttura della pagina

- `.nav` — menu fisso in alto: `chi siamo`, `contatti`, piu' il marchio.
- `.hero` — alta `100vh`, contiene il `<pre>` del render e l'invito a scorrere.
- `.portrait` — due schede impilate, una per persona: ritratto ASCII a sinistra,
  nome e ruolo di fianco. Sotto i 760px le colonne si impilano e tutto si centra.
- `.contact` — riga divisoria, email, una riga di testo, footer. Nient'altro.

Gli strati CRT sono `position:fixed`, quindi coprono anche la sezione contatti: il
testo sta *dentro* il monitor, non sopra. Per questo `.contact` ha `z-index:1` e gli
strati `z-index:2`.

## Il ritratto: ASCII sopra, foto sotto

Sotto la scritta ci sono due **schede**, una per persona:

| scheda | arte | foto |
|---|---|---|
| Andrea Romeo — Consulente AI & Data Platform | `Andrea.txt` (44x24) | `foto2.jpeg` |
| Martino Guggiari — Business Tech Consultant | `martino.txt` (90x49) | `foto1.jpeg` |

Ognuna e' un ritratto ASCII a sinistra e il nome di fianco, nello stesso fosforo
dell'email in fondo. Stesso `data-w` per tutte e due (320px), quindi i due riquadri
sono identici e i nomi cadono esattamente in colonna.

Le due arti hanno pero' griglie molto diverse — 44 colonne per 24 righe contro 90
per 49 — e questo si vede: a parita' di larghezza i caratteri di Andrea vengono il
doppio piu' grossi. Le **proporzioni** invece si somigliano quasi al centesimo: alte
`24/(44*0.6) = 0.909` e `49/(90*0.6) = 0.907` volte la base. I due riquadri vengono
della stessa misura da soli, senza doverla imporre in CSS.

Il ritratto prende il 40% della riga e si ferma al tetto scritto in `data-w`.
`fit()` non puo' misurare la sua colonna — `.col-art` e' larga quanto il `<pre>`, e
il `<pre>` lo stiamo dimensionando proprio in quel momento: misura la scheda intera
e ne prende una frazione.

Il codice e' scritto una volta sola: `setupCard(card)` legge `data-art`,
`data-photo` e `data-w` e monta tutto. Aggiungere una terza persona vuol dire un
`<article class="card">` e un `<script type="text/plain">` con l'arte, niente JS.

**Ogni scheda ha bisogno del suo id di maschera**, e per questo la `<mask>` viene
costruita in JS invece che stare nell'HTML: due `<mask>` con lo stesso nome e la
seconda foto verrebbe ritagliata dalle gocce della prima. Il filtro `#goo`, invece,
e' condiviso: e' identico per tutte. Anche la fase di partenza del giro libero e'
diversa per scheda, se no i due ritratti vagherebbero all'unisono e si vedrebbe
subito che e' un'animazione.

### Le gocce

Il punto e' che **la rivelazione non sta incollata al cursore**. Il mouse fa cadere
una goccia ogni `SPACING` pixel di strada; da quel momento la goccia e' per conto
suo — resta dove e' caduta, si allarga, sta un po', si riassorbe in circa due
secondi. Quello che si vede segue il mouse perche' e' li' che le gocce nascono, ma
con la sua strada: dietro resta una scia che si chiude piano, e la zona scoperta si
spezza in chiazze staccate invece di essere una macchia sola.

Sotto al cursore c'e' pero' una **goccia di testa**, una volta e mezzo piu' grande,
che insegue con una molla molle (rate 9). Senza, muovendo piano non nascerebbe
niente e la foto sembrerebbe non rispondere; ed e' quel ritardo a far sembrare la
cosa liquida invece che agganciata al cursore.

Le gocce sono piccole di proposito (7.5% del lato corto, con raggio randomizzato fra
0.72x e 1.34x): sono la scia e la fusione a fare l'area scoperta, non il singolo
disco. Con gocce grandi si tornava a una macchia sola. Ognuna eredita un filo della
velocita' del mouse, cosi' la scia si sfilaccia in avanti invece di restare una fila
di bolle ferme, e cresce in fretta (primo 16% della vita) per poi scendere lungo una
spalla arrotondata — la forma di una goccia che si asciuga.

### Perche' SVG e non CSS

Le gocce devono **fondersi** fra loro. Sono `<circle>` dentro una `<mask>`, e la
maschera si applica a un `<image>`; il gruppo dei cerchi passa da un filtro in
quattro passaggi:

| | |
|---|---|
| `feGaussianBlur` | sfoca il gruppo: due gocce vicine si toccano nell'alone anche se i dischi non si toccano |
| `feColorMatrix` | ricalca l'alfa a gradino (x26, poi traslata): sopra soglia torna pieno, sotto sparisce |
| `feTurbulence` | una nuvola di rumore, con la frequenza che oscilla su 16s |
| `feDisplacementMap` | la usa per increspare il contorno |

I primi due sono il trucco delle **metaball**: lo sfocato ridiventa un bordo netto,
ma dove c'erano due aloni sovrapposti ora c'e' una forma sola, con la strozzatura in
mezzo. Con dei `div` in CSS ogni goccia resterebbe un disco separato, ed e' per
questo che la lente CSS della versione precedente non poteva funzionare.

Gli ultimi due danno le **ondine**: siccome `baseFrequency` oscilla, la nuvola si
riscala e il profilo delle gocce ondeggia da solo anche a mouse fermo.

### Quando il mouse non c'e'

La pagina si inventa il puntatore: una lissajous lenta con due frequenze non in
rapporto semplice, quindi il giro non si ripete mai uguale e non si legge come un
ciclo. Le gocce cadono esattamente come sotto il mouse vero — stessa funzione,
stesso codice. Parte 1.6s dopo che il mouse esce, e alla prima mossa vera si spegne
passando il comando all'utente.

E' anche la risposta a "come faccio a far capire che si usa il mouse": si vede il
gesto invece di doverlo indovinare. Resta comunque una riga di testo sotto la
cornice, `-> muovi il mouse sull'immagine`, che sparisce alla prima mossa; con
`(hover:none)` diventa "tocca l'immagine".

### Costo

Pool fisso di 34 cerchi riusati a rotazione: nessun nodo creato o distrutto a
runtime, e si tocca il DOM solo quando un raggio cambia davvero (soglia 0.15px). Il
giro libero va solo mentre la sezione e' in vista (`IntersectionObserver`, soglia
`0.35`), e il loop si stacca del tutto quando non c'e' piu' niente di vivo:
sfocatura e turbolenza si ridisegnano a ogni frame e non ha senso tenerle accese a
vuoto. Con `prefers-reduced-motion` il giro non parte e la turbolenza viene congelata
con `pauseAnimations()`.

## Un metodo scartato: il tamburo

Per ottenere i 360° ho prima provato ad avvolgere la parola su un **cilindro**
(settore di 70-150°, lettere in rilievo fra raggio interno ed esterno, intersezione
analitica dei due cilindri). Funzionava, ma va peggio e vale la pena sapere perché:

- La **circonferenza di ingombro** deve stare nel quadro, mentre la parola ne occupa
  solo la corda: si spreca il 35-50% della larghezza utile.
- Ogni lettera scendeva da ~13 a ~8 caratteri di larghezza. A quella risoluzione le
  bande di tono diventano puntinato e la parola non si legge più.
- Agli estremi dell'arco la superficie è quasi di taglio: le prime e le ultime
  lettere si schiacciavano.
- Costava 0.55 ms/frame contro 0.08.

La lastra piatta con rotazione libera dà la stessa libertà di movimento tenendo il
doppio della risoluzione.

## Note e limiti

- **Il pitch è limitato da una relazione precisa.** Ruotando in verticale, la faccia
  superiore del solido diventa alta `DEPTH × sin(pitch)`, mentre le aste delle lettere
  sono spesse 1 unità. Se la fascia supera troppo l'asta, la sostituisce e la lettera
  si spezza. Con `DEPTH` 6 a 11.5° la fascia è 1.19 unità e i tratti reggono ancora —
  verificato ai due estremi. Più in là si perde la forma. Se alzi `MAX_PITCH`, abbassa
  `DEPTH` in proporzione.
- **Luce di rimbalzo dal basso** (`if (my < 0) lam -= my * 0.34`): senza, guardando
  l'oggetto da sotto le facce inferiori sprofondavano nel nero e i tratti orizzontali
  sembravano buchi.
- **I limiti di rotazione non sono scritti a mano.** Si mette un osservatore a
  distanza `ROOM` davanti allo schermo e si orienta la faccia frontale verso di lui;
  l'escursione massima è quindi l'angolo sotto cui si vede il bordo dello schermo da
  quella distanza. Con `ROOM = 2.2` viene **±24.4° in orizzontale e ±15.9° in
  verticale** — abbastanza da leggersi come volume, poco abbastanza da restare sempre
  leggibile. Alza `ROOM` per allontanare l'osservatore e contenere i movimenti.
- **L'inversione è in forma chiusa**, niente ricerca numerica: la normale frontale
  ruotata vale `(-sin(yaw)·cos(pitch), sin(pitch), -cos(yaw)·cos(pitch))`, la si
  uguaglia alla direzione voluta e si ricava `pitch = asin(dy)`,
  `yaw = atan2(-dx, -dz)`. Errore di puntamento misurato: 8·10⁻⁷ gradi.
- **La faccia posteriore è illuminata come una superficie vera**, non a tono fisso,
  altrimenti oltre i 90° la scritta sprofondava nel nero per metà giro. È però sempre
  tenuta sotto la frontale, così si capisce che è il rovescio.
- La griglia si adatta alla finestra (`layout()`): fino a 160 colonne e 64 righe.
  Il tetto è sceso da 190 perché il bloom costa per glifo.
  Il costo per frame è circa `COLS × ROWS × passi`; se serve più fluido, alza `STEP`.
- Con `prefers-reduced-motion` non c'è comportamento speciale: il loop gira comunque.
  (Da sistemare se serve.)

## Da sistemare

- L'email `hello@guggiarom.com` è un segnaposto.
- Anche "Milano" nel footer è un segnaposto.
