# Capitolo 12 — Visualizzazioni SVG

In questo capitolo analizziamo i due componenti grafici dell'applicazione: `GraficoTorta.jsx` e `GraficoIstogramma.jsx`. Entrambi disegnano direttamente con **SVG puro** (Scalable Vector Graphics), senza librerie esterne come Chart.js o D3. Questo significa che tutta la matematica per calcolare posizioni e forme è scritta a mano in JavaScript.

---

## 12.1 GraficoTorta.jsx — Trigonometria e Path SVG

### Cos'è SVG

SVG (Scalable Vector Graphics) è un formato XML per descrivere grafica vettoriale. A differenza di un `<canvas>` (che lavora pixel per pixel), SVG rappresenta forme geometriche come elementi del DOM:

```xml
<svg viewBox="0 0 200 200">
  <circle cx="100" cy="100" r="80" fill="blue" />
  <path d="M 100 100 L 180 100 A 80 80 0 0 1 100 20 Z" fill="red" />
</svg>
```

React può generare questi elementi JSX esattamente come fa con `<div>` o `<input>`, rendendo possibile costruire grafici dinamici senza librerie.

---

### Preparazione dei Dati

Prima di disegnare qualsiasi cosa, il componente aggrega le spese per membro:

```jsx
const totaliPerMembro = membri.map(m => ({
    nome: m.nome,
    totale: spese
        .filter(s => s.chiPaga_ID === m.id)   // solo le spese di questo membro
        .reduce((acc, s) => acc + s.importo, 0) // somma degli importi
})).filter(m => m.totale > 0);  // esclude chi non ha pagato nulla

const totaleGlobale = totaliPerMembro.reduce((acc, m) => acc + m.totale, 0);
```

Il risultato è un array del tipo:
```js
[
  { nome: "Marco", totale: 45.00 },
  { nome: "Sara",  totale: 30.00 },
  { nome: "Luca",  totale: 25.00 }
]  // totaleGlobale = 100.00
```

---

### Il Sistema di Coordinate SVG

Il `viewBox="0 0 200 200"` definisce uno spazio di lavoro 200×200 unità. Le coordinate hanno l'**asse Y invertito** rispetto alla matematica classica: y=0 è in alto, y=200 è in basso.

```
(0,0) ────────────────→ x
  │    ┌──────────────┐
  │    │              │
  ↓    │  cx=100      │
  y    │  cy=100      │
       │    ·         │
       │              │
       └──────────────┘
              200,200
```

Il centro della torta è definito come:
```js
const cx = 100;  // centro X
const cy = 100;  // centro Y
const r  = 80;   // raggio
```

---

### La Matematica della Torta: da Percentuale ad Arco

Ogni fetta della torta è un **settore circolare**. Per disegnarlo serve conoscere:
1. Il punto di partenza sull'arco (x1, y1)
2. Il punto di fine sull'arco (x2, y2)
3. Se l'arco è maggiore o minore di 180° (`largeArc`)

#### Passo 1 — Calcolo dell'angolo

La percentuale di ogni membro viene convertita in **radianti**:

```
angolo = percentuale × 2π
```

Esempio: Marco ha il 45% → angolo = 0.45 × 2π ≈ 2.83 radianti (≈ 162°)

#### Passo 2 — Punto di partenza

Il componente usa `reduce` per accumulare gli angoli. Ogni fetta inizia dove finisce la precedente:

```js
const angoloCorrente = acc.length > 0
    ? acc[acc.length - 1].angoloFine   // fine della fetta precedente
    : -Math.PI / 2;                    // -π/2 = 12 in punto (in alto)
```

> 📌 **Perché `-Math.PI / 2`?** Nell'asse Y invertito di SVG, l'angolo 0 punta a destra (ore 3). Sottraendo π/2 si ruota di 90° in senso antiorario, così la prima fetta parte dall'alto (ore 12), come in un grafico a torta tradizionale.

#### Passo 3 — Coordinate sull'arco

Le coordinate di un punto su una circonferenza si calcolano con le formule trigonometriche fondamentali:

```
x = cx + r × cos(θ)
y = cy + r × sin(θ)
```

Nel codice:
```js
const x1 = cx + r * Math.cos(angoloCorrente);  // punto di partenza
const y1 = cy + r * Math.sin(angoloCorrente);
const x2 = cx + r * Math.cos(angoloFine);      // punto di fine
const y2 = cy + r * Math.sin(angoloFine);
```

#### Passo 4 — Il flag `largeArc`

Il comando SVG `A` (arc) ha bisogno di sapere se disegnare l'arco "corto" o "lungo" quando i due punti possono essere collegati da due archi diversi:

```js
const largeArc = angolo > Math.PI ? 1 : 0;
//               angolo > 180°?  arco lungo : arco corto
```

Se la fetta occupa più del 50% del cerchio (`angolo > π`), bisogna usare l'arco lungo (1), altrimenti quello corto (0).

---

### Il Path SVG Completo

Tutti questi valori vengono assemblati in una stringa **path SVG**:

```js
path: `M ${cx} ${cy} L ${x1} ${y1} A ${r} ${r} 0 ${largeArc} 1 ${x2} ${y2} Z`
```

Decodifica del comando:

| Comando | Significato                                      |
|---------|--------------------------------------------------|
| `M cx cy` | **Move to** — sposta il pennello al centro       |
| `L x1 y1` | **Line to** — traccia una linea verso il bordo   |
| `A r r 0 largeArc 1 x2 y2` | **Arc** — disegna l'arco circolare |
| `Z`       | **Close path** — chiude la forma tornando al centro |

Il risultato visivo:

```
     (cx,cy) = centro
        ·
       /|\
      / | \
     /  |  \
    /   |   \      ← la forma è un "spicchio di pizza"
   ·----+----·
  (x1,y1)  (x2,y2)
  (punto    (punto
   inizio)   fine)
```

---

### Esempio Numerico Completo

Consideriamo 3 membri con totali: Marco 45€, Sara 30€, Luca 25€ (totale 100€).

**Fetta 1 — Marco (45%)**
```
percentuale  = 45/100 = 0.45
angolo       = 0.45 × 2π = 2.827 rad
angoloStart  = -π/2 = -1.571 rad  (ore 12)
angoloFine   = -1.571 + 2.827 = 1.256 rad

x1 = 100 + 80 × cos(-1.571) ≈ 100 + 0   = 100.0
y1 = 100 + 80 × sin(-1.571) ≈ 100 - 80  =  20.0  ← punto ore 12

x2 = 100 + 80 × cos(1.256)  ≈ 100 + 24.7 = 124.7
y2 = 100 + 80 × sin(1.256)  ≈ 100 + 75.6 = 175.6

largeArc = 2.827 > π (3.14)? → NO → 0

path: "M 100 100 L 100 20 A 80 80 0 0 1 124.7 175.6 Z"
```

**Fetta 2 — Sara (30%)** inizia da angoloFine di Marco (1.256 rad) e così via.

---

### Il Donut (Cerchio Bianco al Centro)

La torta è in realtà un **donut chart**: sopra le fette viene disegnato un cerchio bianco che crea l'effetto "buco centrale":

```jsx
<circle cx={cx} cy={cy} r={38} fill="white" />
<text x={cx} y={cy - 6} textAnchor="middle" fontSize="10" fill="#6b7280">
    TOTALE
</text>
<text x={cx} y={cy + 10} textAnchor="middle" fontSize="13" fill="#111827">
    €{totaleGlobale.toFixed(0)}
</text>
```

Il cerchio bianco (raggio 38) è più piccolo del cerchio della torta (raggio 80), creando l'anello visivo. Il testo è centrato con `textAnchor="middle"`.

---

### Schema Visivo del Calcolo

```
          ↑ y=0 (alto)
          |
    ------·------ y=cy=100 (centro)
    x=0   |   x=200
          |
          ↓ y=200 (basso)

  Partenza: θ = -π/2 (ore 12)
  
  θ aumenta in senso orario (asse Y invertito):
  
         -π/2 (ore 12)
           ↑
  π ←------·------→ 0    (ore 9 ← · → ore 3)
           ↓
         +π/2 (ore 6)
```

---

## 12.2 GraficoIstogramma.jsx — Barre Proporzionali

### Struttura del Sistema di Coordinate

L'istogramma usa un sistema di coordinate con **padding** attorno all'area di disegno, per lasciare spazio a etichette e assi:

```js
const svgW          = 500;   // larghezza totale SVG
const svgH          = 260;   // altezza totale SVG
const paddingLeft   = 50;    // spazio per le etichette Y
const paddingRight  = 20;
const paddingTop    = 20;    // spazio per i valori sopra le barre
const paddingBottom = 50;    // spazio per i nomi sotto le barre

const chartW = svgW - paddingLeft - paddingRight;  // 430px area utile
const chartH = svgH - paddingTop - paddingBottom;  // 190px area utile
```

```
┌─────────────────────────────────────────────┐  ← y=0
│         paddingTop (20px)                   │
│  ┌──────────────────────────────────────┐   │  ← y=20
│  │                                      │   │
│p │         AREA GRAFICO                 │   │
│L │         (chartW × chartH)            │   │
│e │                                      │   │
│f │                                      │   │
│t │                                      │   │
│  └──────────────────────────────────────┘   │  ← y=210
│         paddingBottom (50px)                │
└─────────────────────────────────────────────┘  ← y=260
← 50 →←────── chartW=430 ──────────────→← 20 →
```

---

### Calcolo della Larghezza e Posizione delle Barre

```js
const barWidth   = Math.min(50, (chartW / totaliPerMembro.length) * 0.5);
const barSpacing = chartW / totaliPerMembro.length;
```

- `barSpacing` divide l'area in slot uguali per ogni membro
- `barWidth` è il 50% dello slot, con un massimo di 50px (per non avere barre troppo larghe con pochi membri)
- `Math.min(50, ...)` garantisce che con 1-2 membri le barre non occupino tutto lo spazio

**Esempio con 3 membri:**
```
chartW = 430
barSpacing = 430 / 3 ≈ 143px per slot
barWidth   = min(50, 143 × 0.5) = min(50, 71.5) = 50px
```

---

### Calcolo dell'Altezza di Ogni Barra

Le barre sono **proporzionali al massimo**: la barra più alta occupa tutta l'altezza disponibile (`chartH`), le altre sono scalate di conseguenza:

```js
const maxValore = Math.max(...totaliPerMembro.map(m => m.totale));

// per ogni membro:
const barH = (m.totale / maxValore) * chartH;
```

**Esempio:**
```
maxValore = 45€ (Marco)
chartH    = 190px

Marco (45€): barH = (45/45) × 190 = 190px  ← barra piena
Sara  (30€): barH = (30/45) × 190 ≈ 127px
Luca  (25€): barH = (25/45) × 190 ≈ 106px
```

---

### Posizionamento della Barra nel SVG

Poiché in SVG l'asse Y è invertito (y=0 in alto), una barra **parte dal basso** (asse X) e cresce verso l'alto. Il punto `y` del rettangolo è quindi:

```js
const x = paddingLeft + i * barSpacing + barSpacing / 2 - barWidth / 2;
//        ↑ offset sinistro  ↑ slot i-esimo  ↑ centra nel slot  ↑ centra la barra

const y = paddingTop + chartH - barH;
//        ↑ offset superiore  ↑ fondo area  ↑ sale verso l'alto
```

**Visualizzazione:**
```
y=20 (paddingTop)  ┌─────────────────────────┐
                   │                         │
                   │    y = paddingTop        │
                   │      + chartH - barH    │
                   │      ↓                  │
                   │    ┌───┐  ┌───┐  ┌───┐ │
                   │    │   │  │   │  │   │ │
                   │    │   │  │   │  │   │ │  ← barre
                   │    └───┘  └───┘  └───┘ │
y=210 (asse X)     └─────────────────────────┘
```

Il `<rect>` SVG ha attributo `y` che indica il **bordo superiore** del rettangolo, non il basso. Quindi per una barra di altezza 127px il cui fondo deve stare a y=210:
```
y = 210 - 127 = 83
```
Il rettangolo parte da y=83 e scende 127px fino a y=210. ✓

---

### Le Linee di Griglia

Le linee orizzontali di riferimento sono calcolate per i valori 25%, 50%, 75% e 100% del massimo:

```js
const gridLines = [0.25, 0.5, 0.75, 1].map(p => ({
    y:     paddingTop + chartH - chartH * p,   // posizione verticale
    label: `€${(maxValore * p).toFixed(0)}`    // etichetta (es. "€22", "€45")
}));
```

Con `maxValore = 45€` e `chartH = 190`:

| Percentuale | y                        | Etichetta |
|-------------|--------------------------|-----------|
| 25%         | 20 + 190 - 47.5 = 162.5  | €11       |
| 50%         | 20 + 190 - 95   = 115    | €22       |
| 75%         | 20 + 190 - 142.5 = 67.5  | €34       |
| 100%        | 20 + 190 - 190  = 20     | €45       |

---

### Truncation dei Nomi Lunghi

I nomi troppo lunghi vengono troncati per non sovrapporre le etichette:

```js
{m.nome.length > 8 ? m.nome.slice(0, 8) + '…' : m.nome}
// "Alessandro" (10 char) → "Alessand…"
// "Marco"      (5 char)  → "Marco"
```

---

### Confronto GraficoTorta vs GraficoIstogramma

| Aspetto              | `GraficoTorta`                      | `GraficoIstogramma`                  |
|----------------------|--------------------------------------|--------------------------------------|
| Tipo di grafico      | Donut chart                          | Bar chart                            |
| Matematica usata     | Trigonometria (sin, cos), radianti   | Proporzioni lineari                  |
| Primitive SVG        | `<path>` con comando `A` (arc)       | `<rect>`, `<line>`, `<text>`         |
| Sistema coordinate   | Centro (cx, cy) + raggio             | Padding + chartW/chartH              |
| Informazione mostrata| Proporzione di chi ha pagato di più  | Importo assoluto per membro          |
| Sfida principale     | Calcolo angoli cumulativi + largeArc | Inversione asse Y per le barre       |
| Colori               | 8 colori (indigo, blue, green, …)    | 4 colori (red, purple, cyan, orange) |

---

## Perché SVG Puro invece di una Libreria?

Il progetto sceglie di non usare Chart.js, D3 o Recharts per tre motivi:

1. **Nessuna dipendenza aggiuntiva** — il bundle rimane leggero
2. **Controllo totale** — ogni pixel del grafico è customizzabile
3. **Valore didattico** — dimostra la comprensione della matematica sottostante

Il trade-off è che la gestione di casi edge (fette molto piccole, nomi lunghi, valori uguali) richiede più codice rispetto all'uso di una libreria che li gestisce automaticamente.

---

## Riepilogo Formule Chiave

### GraficoTorta
```
percentuale  = totale_membro / totale_globale
angolo_rad   = percentuale × 2π
x            = cx + r × cos(θ)
y            = cy + r × sin(θ)
largeArc     = angolo_rad > π ? 1 : 0
path         = "M cx cy L x1 y1 A r r 0 largeArc 1 x2 y2 Z"
```

### GraficoIstogramma
```
barH         = (totale_membro / max_valore) × chartH
x_barra      = paddingLeft + i × barSpacing + barSpacing/2 - barWidth/2
y_barra      = paddingTop + chartH - barH
y_gridLine   = paddingTop + chartH - chartH × percentuale_griglia
```
