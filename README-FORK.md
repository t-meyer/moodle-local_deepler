# Deepler – Fork der NP Nüsse Arbeitssicherheit GmbH

Dieses Repository ist ein Fork von [brunobaudry/moodle-local_deepler](https://github.com/brunobaudry/moodle-local_deepler).

Die vollständige Plugin-Dokumentation steht unverändert in der [README.md](README.md). Dieses Dokument beschreibt
**ausschließlich, was in diesem Fork vom Upstream abweicht**.

| | |
|---|---|
| **Upstream** | `brunobaudry/moodle-local_deepler` |
| **Fork-Basis** | Commit `f07d435`, Release **v1.9.9.3** |
| **Plugin-Version** | `2026042001` / `v1.9.9.3` – **unverändert gegenüber Upstream** |
| **Upstream-PR** | [#122 – fix: chunking payload size and inline image tokenisation](https://github.com/brunobaudry/moodle-local_deepler/pull/122) (offen) |

---

## Warum dieser Fork existiert

Beim Übersetzen von bild- und HTML-lastigen Kursinhalten traten drei Probleme auf, die im Upstream noch nicht
behoben sind:

1. **HTTP 413 „Payload Too Large"** von der DeepL-API, obwohl das Plugin die Anfragen bereits in Chunks aufteilt.
2. **Die Escaping-Funktionen für LaTeX und `<pre>`-Blöcke waren funktionslos** – die Schalter in den
   erweiterten Einstellungen hatten schlicht keine Wirkung.
3. **Englisch fehlte in der Sprachauswahl**, obwohl ein englisches Sprachpaket installiert war.

Alle Änderungen dieses Forks adressieren diese drei Punkte. Es gibt **keine** organisationsspezifischen
Anpassungen (kein Branding, keine hart verdrahteten Nüsse-Konfigurationen) – der Fork ist so gehalten,
dass er vollständig an den Upstream zurückfließen kann.

---

## Übersicht der Änderungen

### 1. Chunk-Größe wird URL-kodiert gemessen (Fix für HTTP 413)

**Datei:** `classes/external/deeplapi_trait.php`

Die DeepL-PHP-Bibliothek sendet ihre Anfragen als `application/x-www-form-urlencoded`. Das Plugin hat die
Chunk-Größe jedoch anhand der **rohen** UTF-8-Bytes berechnet. Bei HTML-Inhalten weicht das massiv ab:
ein `<` wird zu `%3C` und wächst damit von 1 auf 3 Bytes. Ein Chunk, der laut Rechnung 90 000 Bytes groß war,
konnte real 250 000 Bytes im HTTP-Body belegen – deutlich über DeepLs Hard Limit von 131 072 Bytes (128 KiB).

```diff
- $basebytes = strlen(mb_convert_encoding($basepayload, 'UTF-8')) + $bufferbytes;
+ $basebytes = strlen(urlencode($basepayload)) + $bufferbytes;
...
- $textbytes = strlen(mb_convert_encoding($item['text'], 'UTF-8'));
+ $textbytes = strlen(urlencode($item['text']));
```

Zusätzlich sind die beiden vorher hart codierten Grenzwerte (`100000` und `1024 * 16`) jetzt Admin-Einstellungen
(siehe Punkt 2). Beide werden gegen unsinnige Werte abgesichert – `max(1000, …)` bzw. `max(0, …)` – damit ein
negativer Konfigurationswert die Elvis-Operator-Rückfallebene nicht umgehen kann.

Der `@todo MDL-0000`-Kommentar im Upstream, der genau diese Änderung ankündigte, ist damit erledigt und entfernt.

### 2. Drei neue Admin-Einstellungen

**Dateien:** `settings.php`, `lang/en/local_deepler.php`

Unter *Website-Administration → Plugins → Lokale Plugins → DeepL Translator*:

| Einstellung | Typ | Default | Zweck |
|---|---|---|---|
| `maxchunkbytes` | Zahl | `100000` | Maximale URL-kodierte Bytes pro DeepL-Anfrage. DeepLs Hard Limit sind 131 072 Bytes; der Default lässt Puffer für den HTTP-Envelope. |
| `maxchunkbuffer` | Zahl | `16384` | Pro Chunk reservierte Bytes für statische Parameter (`target_lang`, Optionen usw.). |
| `stripimagesadmin` | Checkbox | aktiviert | Voreinstellung des Schalters „Inline-Base64-Bilder tokenisieren" auf der Übersetzungsseite. |

> **Hinweis zum Update-Verhalten:** Da `version.php` bewusst **nicht** hochgezählt wurde (der Fork soll
> versionskompatibel zum Upstream bleiben), löst ein reines Datei-Update keinen Moodle-Plugin-Upgrade aus.
> Dadurch werden die Admin-Defaults nicht automatisch in die Config geschrieben. Für `maxchunkbytes` und
> `maxchunkbuffer` ist das unkritisch – der Code hat identische Fallback-Werte im Quellcode. Der Default
> von `stripimagesadmin` greift jedoch erst, **nachdem die Einstellungsseite einmal gespeichert wurde**;
> vorher startet der Schalter auf der Übersetzungsseite deaktiviert. Nach der Installation also einmal
> die Plugin-Einstellungen aufrufen und speichern.

### 3. Reparatur der `escapePatterns`-Pipeline (stiller Upstream-Bug)

**Dateien:** `amd/src/local/translation.js`, `amd/src/local/ui_deepler.js`

In `translation.js` existierte ein Modul-weites `let escapePatterns = {}`, das **nie befüllt wurde**. Es wurde
an `Tokeniser.preprocess()` durchgereicht, wodurch der Tokeniser konsistent zu dem Ergebnis kam, dass nichts
zu escapen sei. Konsequenz: Die Schalter *„Escape LaTeX"* und *„Escape PRE"* in den erweiterten Einstellungen
waren im Upstream wirkungslos – LaTeX-Formeln und `<pre>`-Blöcke wurden trotz aktivierter Option an DeepL gesendet.

Die Variable ist entfernt. `escapePatterns` wird jetzt in `callDeeplServices()` aus dem aktuellen UI-Zustand
gebaut und explizit als Parameter an `initTempForKey()` übergeben:

```javascript
const escapePatterns = {
    PRETAG: settingsUI[Selectors.actions.escapePre]?.checked ?? false,
    LATEX: settingsUI[Selectors.actions.escapeLatex]?.checked ?? false,
    DATAURI: settingsUI[Selectors.actions.stripImages]?.checked ?? false,
};
```

### 4. Neues `DATAURI`-Token für Inline-Base64-Bilder

**Dateien:** `amd/src/local/tokeniser.js`, `amd/src/local/selectors.js`,
`templates/translate_page.mustache`, `classes/output/translate_page.php`

Über den Moodle-Editor eingefügte Bilder landen häufig als `<img src="data:image/png;base64,…">` im Feld.
Ein einziges solches Bild kann mehrere hundert Kilobyte an Base64-Text erzeugen – Binärdaten, die für die
Übersetzung völlig nutzlos sind, den Payload aber zuverlässig über DeepLs Limit treiben.

Der Tokeniser kennt daher ein drittes Muster neben `PRETAG` und `LATEX`:

```javascript
{regex: /<img\b[^>]*?src=(?:"data:[^"]*"|'data:[^']*')[^>]*?>/gs, type: 'DATAURI'}
```

Passende `<img>`-Tags werden vor dem Versand durch opake Tokens ersetzt und nach der Antwort unverändert
zurückgeschrieben. Steuerbar über einen neuen Schalter **„Strip inline base64 images"** im Panel
*Other Settings* der Übersetzungsseite; die Voreinstellung kommt aus `stripimagesadmin`.

### 5. Sprachauswahl: Basissprachen gegen regionale DeepL-Varianten matchen

**Datei:** `classes/local/services/lang_helper.php`

DeepL liefert für Englisch ausschließlich die Zielcodes `EN-US` und `EN-GB`; Moodle installiert dagegen das
Sprachpaket als schlichtes `en`. Der Kompatibilitätscheck prüfte nur eine Richtung – Moodle-Code beginnt mit
DeepL-Code – und übersah damit den umgekehrten Fall, in dem ein kurzer Moodle-Code zu einem längeren
DeepL-Code passen muss. Englisch (und ebenso `pt` gegenüber `PT-BR` / `PT-PT`) fiel deshalb aus der Auswahlliste.

```diff
- if ($deepl === $moodle || str_starts_with($moodle, $deepl . '-')) {
+ if ($deepl === $moodle || str_starts_with($moodle, $deepl . '-') || str_starts_with($deepl, $moodle . '-')) {
```

### 6. Test

**Datei:** `tests/external/gettranslation_test.php`

Neu: `test_chunkpayload_urlencode_expansion()`. Zwei Items mit je 34 000 `<`-Zeichen ergeben URL-kodiert
exakt 102 000 Bytes pro Item und müssen daher – anders als bei der alten Rohbyte-Messung – jeweils in einem
eigenen Chunk landen. Abgedeckt sind `get_translation::chunk_payload` und `get_rephrase::chunk_payload`.

### 7. Build und Housekeeping

- `amd/build/local/{tokeniser,selectors,translation,ui_deepler}.min.js` (+ Source Maps) neu gebaut,
  via terser mit expliziten AMD-Modul-IDs.
- `.gitignore`: `docs/` ausgeschlossen, damit interne Planungsdateien nicht versehentlich committet werden.

---

## Vollständige Dateiliste

```
.gitignore                              |  2 ++
amd/build/local/selectors.min.js        |  (rebuild)
amd/build/local/tokeniser.min.js        |  (rebuild)
amd/build/local/translation.min.js      |  (rebuild)
amd/build/local/ui_deepler.min.js       |  (rebuild)
amd/src/local/selectors.js              |  1 +
amd/src/local/tokeniser.js              |  3 ++-
amd/src/local/translation.js            |  4 ++--
amd/src/local/ui_deepler.js             | 11 ++++++++++-
classes/external/deeplapi_trait.php     |  9 ++++-----
classes/local/services/lang_helper.php  |  2 +-
classes/output/translate_page.php       |  1 +
lang/en/local_deepler.php               | 13 +++++++++++++
settings.php                            | 28 ++++++++++++++++++++++++++++
templates/translate_page.mustache       |  6 ++++++
tests/external/gettranslation_test.php  | 30 ++++++++++++++++++++++++++++
```

Nicht berührt: Datenbankschema (`db/`), Capabilities, Webservice-Definitionen, Glossar- und
Token-Verwaltung. Es ist **keine Migration** erforderlich.

Übersetzungen der neuen Sprachstrings liegen bisher nur in `lang/en/` vor.

---

## Upstream-Sync

Beide Remotes einrichten:

```bash
git remote add origin   https://github.com/NP-Nusse-Arbeitssicherheit-GmbH/moodle-local_deepler.git
git remote add upstream https://github.com/brunobaudry/moodle-local_deepler.git
```

Änderungen vom Upstream übernehmen:

```bash
git fetch upstream
git merge upstream/main        # oder: git rebase upstream/main
```

Sollte PR #122 im Upstream gemergt werden, lösen sich die Punkte 1–6 dieses Dokuments beim nächsten Sync
von selbst auf. Dieses Dokument sollte dann entsprechend gekürzt oder entfernt werden.

---

## Lizenz

Unverändert **GNU GPL v3 or later**, wie im Upstream. Siehe [LICENSE.md](LICENSE.md).

Ursprüngliche Urheber: Kaleb Heitzman (2022), Bruno Baudry (2024).
