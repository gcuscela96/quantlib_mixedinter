# Fork QuantLib: interpolazione mista con "data di frontiera"

Questo fork aggiunge a QuantLib un nuovo schema di interpolazione per curve
dei tassi bootstrappate:

* **prima della data di frontiera**: interpolazione lineare in `r*t`
  (equivalente, per costruzione, all'interpolazione log-lineare del
  discount factor `exp(-r t)`  — è la convenzione di mercato standard
  "linear in r*t" / "log-linear discount");
* **dalla data di frontiera in poi**: spline cubica naturale (derivata
  seconda nulla ai due estremi), costruita usando **esclusivamente** i
  punti a partire dallo strumento di frontiera in poi — i punti precedenti
  non influenzano in alcun modo la forma dello spline.

Il punto di frontiera è condiviso da entrambi i pezzi (è l'ultimo nodo del
tratto lineare e il primo nodo dello spline cubico), quindi la curva
risultante è per costruzione continua (C0) nel punto di frontiera, anche se
in generale non derivabile lì (derivata prima e seconda possono avere un
salto, come è naturale passando da un tratto lineare-in-rt a uno cubico).

## File aggiunti

- `ql/math/interpolations/frontierlinearrtcubicinterpolation.hpp`
  Nuova classe di interpolazione (header-only, nessuna modifica ai file
  esistenti di QuantLib). Definisce:
  - `FrontierLinearRtCubicInterpolation` — la classe `Interpolation` vera
    e propria (parametri cubici configurabili: Spline/Kruger/monotonic/
    condizioni al contorno, come nella `CubicInterpolation` di QuantLib).
  - `FrontierLinearRtCubicNaturalSpline` — comodo alias con spline
    naturale (Spline, non monotona, `SecondDerivative=0` ai due estremi).
  - `FrontierLinearRtCubic` / `FrontierLinearRtCubicNatural` — le
    "traits"/factory da passare come `Interpolator` a
    `QuantLib::PiecewiseYieldCurve<Traits, Interpolator>` (tipicamente con
    `Traits = ZeroYield`, dato che l'interpolazione qui è pensata per
    agire su coppie `(tempo, tasso zero)`).
- `Examples/FrontierMixedCurve/frontier_mixed_curve_example.cpp`
  Esempio standalone che bootstrappa una curva fittizia con frontiera a
  2 anni e stampa tasso zero e `r*t` lungo tutta la curva.

Nessun file esistente di QuantLib è stato modificato: si tratta di
un'aggiunta puramente additiva, quindi il merge/rebase su versioni più
recenti di QuantLib upstream dovrebbe restare semplice.

## Perché funziona anche durante il bootstrap iterativo

`PiecewiseYieldCurve` costruisce la curva un pilastro alla volta
(`IterativeBootstrap`): ad ogni passo richiama
`interpolator_.interpolate(times.begin(), times.begin()+i+1, data.begin())`
su un array che cresce progressivamente. Una interpolazione mista con un
indice di frontiera "fisso" (come la `MixedLinearCubic` già presente in
QuantLib) fallirebbe nei primi passi, quando l'array è ancora più corto
dell'indice di frontiera.

Per questo `FrontierLinearRtCubicInterpolationImpl`:

1. non memorizza un indice fisso, ma un **tempo** di frontiera
   (`frontierTime`, espresso nella stessa unità di tempo/day counter della
   curva);
2. ad ogni chiamata ricalcola dinamicamente, sull'array corrente
   (qualunque sia la sua lunghezza in quel momento del bootstrap),
   l'indice `n` dell'ultimo punto con `t <= frontierTime`;
3. se dopo la frontiera non ci sono ancora abbastanza punti per costruire
   uno spline cubico sensato (meno di 3, frontiera inclusa), il
   costruttore lancia un'eccezione **nel costruttore stesso** (prima di
   `update()`);
4. `IterativeBootstrap` (già in QuantLib, invariato) intercetta quella
   eccezione e, siccome l'interpolatore dichiara `global = true`, ricade
   automaticamente su una semplice interpolazione lineare per quel passo
   intermedio — esattamente lo stesso meccanismo che QuantLib usa già per
   `Cubic`, `LogCubic`, `MixedLinearCubic`, ecc. Non è stata quindi
   necessaria alcuna modifica a `iterativebootstrap.hpp`.

Il risultato: durante i primi passi del bootstrap (prima che la curva
raggiunga/superi la data di frontiera) si usa una scaffolding lineare
provvisoria; una volta che il bootstrap arriva ad avere abbastanza punti
oltre la frontiera, la vera interpolazione mista prende il sopravvento, e
lo spline finale è quello descritto sopra, indipendentemente dall'ordine
in cui i pilastri sono stati aggiunti.

## Uso in C++

```cpp
#include <ql/math/interpolations/frontierlinearrtcubicinterpolation.hpp>
#include <ql/termstructures/yield/piecewiseyieldcurve.hpp>

using namespace QuantLib;

Date today = ...;
DayCounter dc = Actual365Fixed();
Date frontierDate = calendar.advance(today, 2*Years);
Time frontierTime = dc.yearFraction(today, frontierDate);

FrontierLinearRtCubicNatural interpolator(frontierTime);

typedef PiecewiseYieldCurve<ZeroYield, FrontierLinearRtCubicNatural> Curve;
Curve curve(today, helpers, dc, interpolator);
```

Se si vogliono condizioni al contorno diverse dalla spline naturale pura
(es. Kruger, monotona, o `FirstDerivative`/`NotAKnot`), si usa invece la
classe generale `FrontierLinearRtCubic`, che espone tutti i parametri di
`CubicInterpolation`:

```cpp
FrontierLinearRtCubic interpolator(
    frontierTime,
    CubicInterpolation::Spline, /*monotonic=*/false,
    CubicInterpolation::SecondDerivative, 0.0,
    CubicInterpolation::SecondDerivative, 0.0);
```

## Come compilare e testare in questo checkout

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DQL_BUILD_TEST_SUITE=OFF -DQL_BUILD_EXAMPLES=OFF ..
make -j$(nproc)
sudo make install   # oppure puntare CMAKE_INSTALL_PREFIX a una cartella locale

g++ -std=c++17 ../Examples/FrontierMixedCurve/frontier_mixed_curve_example.cpp \
    $(pkg-config --cflags --libs QuantLib) -o frontier_example
./frontier_example
```

Verifica attesa: nella colonna `r*t` stampata dall'esempio, i valori
prima della frontiera (2Y) devono crescere in modo esattamente lineare
tra un pilastro e l'altro; dopo la frontiera, la colonna `zero rate` deve
seguire uno spline cubico "liscio" (nessuna spezzata), e i due tratti
devono coincidere esattamente nel punto di frontiera.

## Esporlo in Python (QuantLib-SWIG)

Questa modifica non tocca la libreria C++ esistente: è un header
aggiuntivo, quindi **non richiede nessuna modifica ai binding SWIG
esistenti per compilare**. Per poterla però *usare da Python* servono i
binding per:

1. la classe/trait `FrontierLinearRtCubicNatural` (o `FrontierLinearRtCubic`);
2. una istanza di `PiecewiseYieldCurve<ZeroYield, FrontierLinearRtCubicNatural>`
   esposta con un nome Python (es. `PiecewiseFrontierMixedZeroCurve`).

In `QuantLib-SWIG` (repo separato, `github.com/lballabio/QuantLib-SWIG`),
i file rilevanti sono `SWIG/interpolation.i` (dove sono dichiarate le
classi di interpolazione come `Cubic`, `LogCubic`, `MixedLinearCubic`) e
`SWIG/piecewiseyieldcurve.i` (dove ogni combinazione Traits+Interpolator
concreta viene istanziata con `%template(...)`, seguendo lo stesso schema
usato per `PiecewiseLogCubicDiscount`, `PiecewiseCubicZero`, ecc.).

Questo è già stato fatto in questo fork (vedi `QuantLib-SWIG.patch` nel
pacchetto consegnato): in `SWIG/interpolation.i` sono state aggiunte le
classi `FrontierLinearRtCubic`/`FrontierLinearRtCubicNatural` (stesso
schema usato lì per `MixedLinearCubic`), e in `SWIG/piecewiseyieldcurve.i`
è stata aggiunta la riga

```
export_piecewise_curve(PiecewiseFrontierMixedZero,ZeroYield,FrontierLinearRtCubicNatural);
```

che espone in Python la classe `PiecewiseFrontierMixedZero`.

**Verificato end-to-end in questa sessione**: libreria C++ compilata da
zero (CMake), wheel Python generata con SWIG 4.2 + build isolata, e
testata con uno script Python che bootstrappa una curva con 12 strumenti
fittizi e frontiera a 2 anni. Risultato:

- prima della frontiera, `r*t` cresce esattamente linearmente tra un
  pilastro e l'altro (verificato numericamente);
- dopo la frontiera, il tasso zero segue uno spline cubico naturale
  liscio, senza spezzate;
- nel punto di frontiera `r(frontiera-ε)`, `r(frontiera)` e
  `r(frontiera+ε)` coincidono fino all'ottava cifra decimale (continuità
  C0 confermata numericamente, non solo per costruzione teorica).

Esempio d'uso Python (vedi anche `test_frontier_python.py` nel pacchetto):

```python
import QuantLib as ql

interpolator = ql.FrontierLinearRtCubicNatural(frontier_time)
curve = ql.PiecewiseFrontierMixedZero(today, helpers, dc, [], [], interpolator)
```

## Variante: spline cubica fittata su r*t invece che su r

Di default (`FrontierLinearRtCubicNatural`, classe `FrontierLinearRtCubic` con
`cubicInRt = false`) il tratto cubico dopo la frontiera è fittato
direttamente sui valori `y` (tipicamente il tasso zero). È stata aggiunta
un'opzione, `cubicInRt`, che invece fitta lo spline cubico sui valori
trasformati `y_i * x_i` (r*t) — la stessa trasformazione già usata nel
tratto lineare prima della frontiera — dividendo poi per `x` in fase di
valutazione: `r(x) = C(x) / x` anche per `x > frontierTime`.

Le due varianti coincidono esattamente nei pilastri (l'interpolazione
passa comunque per i punti dati) e nel punto di continuità alla
frontiera, ma danno forme leggermente diverse tra un pilastro e l'altro,
perché lo spline è calcolato su una quantità diversa (r vs r*t).

Classi disponibili:

- `FrontierLinearRtCubicNatural` — invariata, cubico naturale su `y`.
- `FrontierLinearRtCubicNaturalRt` — nuova, cubico naturale su `y*x`.
- `FrontierLinearRtCubic` — trait generale, ora accetta anche il
  parametro `cubicInRt` (default `false`) oltre ai parametri esistenti
  (DerivativeApprox, monotonic, boundary conditions).

In C++:

```cpp
// cubico naturale su r*t invece che su r
FrontierLinearRtCubicNaturalRt interpolator(frontierTime);

// oppure, con controllo esplicito su tutti i parametri:
FrontierLinearRtCubic interpolator2(
    frontierTime,
    CubicInterpolation::Spline, /*monotonic=*/false,
    CubicInterpolation::SecondDerivative, 0.0,
    CubicInterpolation::SecondDerivative, 0.0,
    /*cubicInRt=*/true);
```

In Python, la nuova classe è esposta insieme a una curva dedicata:

```python
interpolator = ql.FrontierLinearRtCubicNaturalRt(frontier_time)
curve = ql.PiecewiseFrontierMixedRtZero(today, helpers, dc, [], [], interpolator)
```

**Verificato in questa sessione**: entrambe le varianti (`PiecewiseFrontierMixedZero`
su `FrontierLinearRtCubicNatural` e `PiecewiseFrontierMixedRtZero` su
`FrontierLinearRtCubicNaturalRt`) sono state bootstrappate sugli stessi
strumenti e confrontate punto per punto — coincidono esattamente nei
pilastri e nel punto di frontiera, differiscono (come atteso) tra un
pilastro e l'altro, e sono entrambe continue (C0) alla frontiera fino
alla decima cifra decimale.

## Helper aggiuntivi per la costruzione di curve: pillar custom, forward
## start fisso, "pagamento anticipato", day count/flat rate custom

Oltre allo schema di interpolazione, il fork aggiunge classi riutilizzabili
per costruire strumenti di bootstrap con caratteristiche non coperte dagli
helper nativi di QuantLib:

- `ql/termstructures/yield/ratehelperwithcustompillar.hpp` —
  `OvernightIborBasisSwapRateHelperCustomPillar`: stesso strumento e
  stessa logica di bootstrap di `OvernightIborBasisSwapRateHelper`
  nativo (spot -> spot+tenor), ma con il *pillar* di bootstrap forzato
  a una data custom (es. una data ECB/IMM) invece che alla naturale
  maturity dello strumento.

- `ql/termstructures/yield/forwardstartingbasisswaphelper.hpp` —
  `OvernightIborBasisSwapRateHelperForwardStarting`: stesso strumento,
  ma con `startDate`/`endDate` *espliciti e fissi* (es. due date
  IMM/ECB), indipendenti dalla evaluation date. Il pillar di default è
  `endDate`, oppure un `customPillarDate` esplicito.

  **`customPaymentDate` ("pagamento anticipato")**: entrambe le gambe
  (overnight e ibor) continuano ad accrue/fixare sul loro vero periodo
  (`startDate` -> `endDate`), ma il *pagamento* di entrambe le cedole
  viene forzato su `customPaymentDate` invece che sulla naturale fine
  accrual — tipico di uno strumento "STUB" il cui accrual/fixing copre
  l'intero periodo reale (es. spot -> spot+3M) ma il cui cash flow è
  pagato prima (es. su una data IMM), per non sovrapporsi al prossimo
  strumento sulla curva. Se `customPillarDate` non è specificato, il
  pillar di bootstrap si posiziona automaticamente su
  `customPaymentDate` (non su `endDate`). Supportato solo per uno
  schedule a un solo periodo (uno `QL_REQUIRE` lo impone: la
  frequenza/il tenor dell'indice deve coprire l'intero periodo
  startDate-endDate in una sola cedola per gamba). Implementato tramite
  `ql/cashflows/earlypaymentcoupon.hpp` (`EarlyPaymentCoupon`): molte
  classi di coupon native (in particolare `OvernightIndexedCoupon`)
  impongono strutturalmente `paymentDate >= accrualEndDate` nel proprio
  costruttore, quindi un "pagamento anticipato" non è costruibile
  passando direttamente una data di pagamento anteriore tramite i
  builder nativi. `EarlyPaymentCoupon` risolve il problema mantenendo
  intatta la cedola "naturale" (con pagamento == fine accrual, quindi
  valida per ogni invariante nativo) e avvolgendola in un wrapper che si
  limita a *ridatare* il cash flow visto da NPV/BPS, delegando importo,
  rate, day counter e accrued amount alla cedola naturale sottostante.
  Nota tecnica: `EarlyPaymentCoupon` sovrascrive anche `deepUpdate()`
  per propagarlo alla cedola sottostante, perché è esattamente il
  meccanismo (`Swap::deepUpdate()`, chiamato da `impliedQuote()` prima
  di ogni NPV) con cui QuantLib invalida la cache di una cedola prima di
  ogni nuova valutazione del bootstrap; senza questo inoltro il tasso
  della cedola avvolta resterebbe agganciato al primo valore calcolato,
  dando un bootstrap "root not bracketed" con funzione obiettivo piatta.

  **`overnightDayCounter` / `iborDayCounter`**: permettono a ciascuna
  gamba di usare un day counter per l'accrual diverso da quello nativo
  del proprio indice (sono semplicemente inoltrati a
  `OvernightLeg::withPaymentDayCounter()` / `IborLeg::withPaymentDayCounter()`,
  gia' nativamente supportati da QuantLib). Lasciarli di default
  (costruiti vuoti) mantiene il day counter nativo dell'indice.

  **`overnightFlatRatePayment` / `iborFlatRatePayment`** ("niente day
  count"): commutano la gamba a un cash flow *flat*, `amount = rate() *
  nominal()`, **senza alcuna** scalatura per accrual period/day count —
  utile per strumenti quotati/regolati come un pagamento "a tasso secco"
  anziché pro-rata. Implementato tramite una nuova classe gemella di
  `EarlyPaymentCoupon`, `ql/cashflows/flatratecoupon.hpp`
  (`FlatRateCoupon`), con lo stesso schema (avvolge la cedola naturale,
  delega tutto tranne l'ammontare). **Attenzione**: `Coupon::accrualPeriod()`
  non è virtuale in QuantLib, quindi non è sovrascrivibile — quando la
  gamba che porta lo spread quotato (overnight, a meno che
  `basisOnIborLeg` sia `true`) usa il pagamento flat,
  `impliedQuote()` calcola la sensitivity al basis point di quella gamba
  *direttamente* (nominale scontato una sola volta alla sua data di
  pagamento) invece che tramite `Swap::legBPS()`, che altrimenti
  moltiplicherebbe ancora per il vero accrual period del coupon
  sottostante, dando un bootstrap errato.

  I due meccanismi (day counter custom, pagamento flat) sono
  indipendenti e combinabili per gamba, oltre che con `customPaymentDate`
  (che resta condiviso da entrambe le gambe).

Esempio Python (caso reale: basis ESTR-comp+spread vs Euribor fixing,
entrambe le gambe senza day count nell'ammontare, stessa data di
pagamento custom per entrambe, bootstrap della curva Euribor):

```python
helper = ql.OvernightIborBasisSwapRateHelperForwardStarting(
    spot_date, real_end_date, basis_quote, calendar, ql.ModifiedFollowing, False,
    estr, euribor3m, estr_handle, False,
    basisOnIborLeg=False,                     # spread quotato sulla gamba ESTR
    customPaymentDate=custom_payment_date,    # stessa data per entrambe le gambe
    overnightFlatRatePayment=True,            # gamba ESTR: amount = (ESTR_comp+spread)*nominal
    iborFlatRatePayment=True,                 # gamba Euribor: amount = fixing*nominal
)
```

**Verificato in questa sessione**: sia in C++ standalone sia in Python
end-to-end (bootstrap completo con `PiecewiseLinearZero`) per tutte le
combinazioni — day counter custom da solo, pagamento flat sulla gamba
overnight (quella con lo spread) insieme a `customPaymentDate`,
pagamento flat sulla sola gamba ibor con la gamba spread invariata:
in ogni caso l'implied quote riprezza esattamente il basis quotato
(diff ~1e-13/1e-15), l'ammontare della gamba flat coincide esattamente
con `rate()*nominal()` (e non con la versione scalata per accrual
period), e il fixing Euribor implicito nella curva in costruzione resta
quello correttamente usato dal bootstrap.
