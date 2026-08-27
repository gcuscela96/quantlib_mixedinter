# Come ottenere la wheel Windows senza installare nulla in locale

Questo pacchetto contiene tutto il necessario per far compilare la wheel
Python (per Windows, Python 3.9) su un runner cloud gratuito di GitHub
Actions. Non serve installare Visual Studio, Boost, SWIG o altro sul tuo
PC: ti serve solo un account GitHub (va bene anche uno personale) e un
browser.

## Passi

1. Crea un nuovo repository GitHub (puoi lasciarlo privato).
2. Carica in quel repository, mantenendo la struttura di cartelle:
   - `QuantLib.patch`
   - `QuantLib-SWIG.patch`
   - `.github/workflows/build-windows-wheel.yml`

   (Il modo più semplice: sul sito GitHub usa "Add file" -> "Upload
   files" e trascina l'intera cartella `github_actions_kit`, oppure se
   hai `git` a disposizione anche solo in locale senza compilatori:
   ```
   git init
   git add .
   git commit -m "add build kit"
   git remote add origin <url-del-tuo-repo>
   git push -u origin main
   ```
   )

3. Vai nella scheda **Actions** del repository su GitHub.
4. Seleziona il workflow "Build Windows QuantLib wheel (frontier fork)"
   e clicca **Run workflow** (è configurato per partire solo manualmente,
   `workflow_dispatch`).
5. Attendi il completamento (indicativamente 20-40 minuti: gran parte
   del tempo se ne va per installare Boost via Chocolatey e per
   compilare l'enorme file generato da SWIG).
6. A fine esecuzione, apri la run completata e scarica l'artifact
   chiamato `quantlib-frontier-windows-wheel`: è uno zip che contiene il
   file `.whl`.
7. Sul tuo PC Windows, con Python 3.9.9 installato:
   ```powershell
   pip install quantlib-1.44.dev0-cp39-abi3-win_amd64.whl
   ```
   (il nome esatto del file può variare leggermente; usa quello che hai
   scaricato).

Da qui in poi l'uso è identico a quanto già mostrato:

```python
import QuantLib as ql

interpolator = ql.FrontierLinearRtCubicNatural(frontier_time)
curve = ql.PiecewiseFrontierMixedZero(today, helpers, dc, [], [], interpolator)
```

## Nota di onestà

Questo workflow non è stato eseguito/testato da me in questa sessione
(non ho un ambiente Windows né un runner GitHub Actions a disposizione
qui): è scritto seguendo il flusso di build standard e documentato di
QuantLib/QuantLib-SWIG su Windows (Visual Studio + Boost via Chocolatey
+ SWIG), ma è possibile che al primo tentativo serva qualche piccolo
aggiustamento (es. nome esatto del pacchetto Boost, versione di Visual
Studio sul runner). Se il job fallisce, copiami il log dello step che è
andato in errore (lo trovi cliccando sulla run fallita su GitHub) e lo
sistemiamo insieme.
