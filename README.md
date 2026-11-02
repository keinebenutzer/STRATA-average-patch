# Strata-Patch: Speed-Anzeige als Lauf-Durchschnitt

Datum: 2026-10-29, neu aufgebaut für **0.1.55** (HEAD c62d5e3a)
Patch-Datei: `speed_average.patch` (git diff, 5 Dateien) · nur Web-GUI: `speed_average_appjs.patch`

## Wichtig: drei verschiedene Oberflächen
Strata hat drei Monitore. Der Patch deckt alle drei ab:
1. **Web-App** `http://host:port/` → `serve/web/index.html` + `serve/web/app.js` (Monitor-Tab, Karte „Speed" mit **Decode** und **Prefill**). Das ist die WebGUI, die man im LAN sieht.
2. **API-Monitor-Seite** `http://host:port/api-monitor` (nur mit `--api-monitor`) → `serve/web/monitor.html` + `monitor.js`.
3. **Terminal-Monitor** (`strata … monitor`) → `strata_cli/overview.py`.

Vorher zeigte jede Oberfläche die Speed des **letzten** Abschlusses. Jetzt zeigt der **Idle-Zustand** den **token-gewichteten Durchschnitt des Laufs** (aus den Server-`totals`). Live-Werte bleiben unverändert.

## Änderungen
### serve/web/app.js  (die WebGUI, die du siehst)
- Idle **Decode**: `totals.output_tokens / (totals.decode_ms/1000)` → Notiz „Decode · run average" (statt „Decode last request").
- Idle **Prefill**: `(totals.prompt_tokens - totals.reused) / (totals.prompt_ms/1000)` → „Prefill · run average" (statt „Prefill last request").
- Live bleibt: Generieren = `live.tok_s` („Decode now"), Lesen = `live.prefill_tok_s_mean` („Prefill now/this request").
- Fällt auf „last request" zurück, wenn keine `totals` (nie eine erfundene 0).

### serve/web/monitor.js + monitor.html  (API-Monitor-Seite)
- `averageDecodeSpeed(rows)`: token-gewichteter Durchschnitt über alle abgeschlossenen Requests.
- Karte „Last decode speed" → **„Average decode speed"**.

### strata_cli/overview.py  (Terminal-Monitor)
- `run_average_speed(metrics)` aus den `totals`; Idle-Kachel → „average of the run". Live („now"/„reading") bleibt.

## Entscheidung (per Rückfrage bestätigt)
- Nur der Idle-Wert wird zum Durchschnitt; live bleibt.
- Durchschnitt token-gewichtet (nicht einfaches Mittel).

## Verifikation im Prüfstand (kein GPU-Lauf)
`uv` (Python 3.12; textual 8.2.8, rich, jinja2, psutil, regex):
- `tests.terminal.test_overview` → OK (neu: `test_idle_shows_the_run_average`, Idle = 40.0 tok/s)
- `serve.test_monitor` → OK (monitor.js ausgeliefert, `textContent` ja, `innerHTML` nein)
- `serve.test_answer_speed` → OK
- Gesamtlauf dieser drei: **30/30 OK**
- `node --check serve/web/app.js` und `.../monitor.js` → Syntax OK
- `git apply --check --reverse` → Patch passt exakt


## Anwenden
```sh
cd /Pfad/zu/strata
git apply /PFAD/ZUM/PATCH/strata_speed_average/speed_average.patch
```
Wichtig: Patch muss in den Baum, aus dem der **laufende** Server bedient wird (`python -c "import serve.server as s; print(s.ROOT)"`). Läuft der Server über eine installierte Kopie (site-packages), dort patchen oder `uv pip install -e .` im gepatchten Baum und Server neu starten. Die Web-Dateien werden pro Request frisch von der Platte gelesen (`Cache-Control: no-cache`) — nach dem Patch genügt ein Browser-Neuladen (Hard-Reload), kein Kompilieren.
