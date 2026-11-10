# Text hinter dem Kopf

Von Paul (Effekt 2/100). Eine Datei, die du deiner KI gibst. Sie baut dir daraus ein kleines Programm,
das einen großen Titel **hinter** dich ins Video setzt: Dein Kopf verdeckt einen Teil der Schrift, und
wenn du dich bewegst, bleibt die Schrift trotzdem hinter dir. Läuft auf deinem Rechner, kostenlos.

## Für dich: so benutzt du die Datei

**Du brauchst:** einen Mac oder Windows-PC und einen KI-Assistenten, der auf deinem Rechner arbeiten
darf, zum Beispiel Claude Code (in der Claude-App der Tab „Code“). In einem normalen Chatfenster geht
es nicht.

1. Leg einen neuen Ordner an und leg diese Datei und dein Video hinein. Hochkant, eine Person, über dem
   Kopf etwas Platz.
2. Öffne den Ordner in Claude Code und schreib: **„Lies TEXT-HINTER-DEM-KOPF.md und setz den Titel
   ‚DEIN WORT‘ hinter meinen Kopf.“**

Beim ersten Mal richtet die KI ein paar Dinge ein, das dauert etwa 5 Minuten. Am schnellsten geht es auf
einem Mac, weil der Mac das Freistellen schon eingebaut hat.

**Tipp beim Filmen:** Ein Drittel Luft über dem Kopf lassen. Und einmal den Kopf drehen, während der
Titel steht: Dann sieht man, dass das kein Aufkleber ist.

---

## Für die KI: Auftrag

Du baust ein wiederverwendbares Skript `kopf.py` und setzt damit einen Titel hinter die Person. Erkläre
jeden Schritt in einem kurzen Satz ohne Fachwörter. Alle Zahlen unten stehen als Einstellungen oben in
`kopf.py`.

### Regeln

- Das Originalvideo wird nie verändert. Alles Neue kommt nach `ausgabe/<Videoname ohne Endung>/`.
- Installiert wird in diesen Ordner: `.venv` und `schriften/`. Fehlt ffmpeg oder Python, gib der Person
  die Zeile aus der Tabelle und warte.

| | Mac (Terminal) | Windows (PowerShell) |
|---|---|---|
| ffmpeg | `brew install ffmpeg` | `winget install --id Gyan.FFmpeg -e` |
| Python 3.12 | `brew install python@3.12` | `winget install --id Python.Python.3.12 -e` |

### Schritt 0: Einrichten

1. Python 3.10, 3.11 oder 3.12 suchen (nicht 3.13 und neuer), `.venv` damit anlegen, `numpy` und
   `pillow` installieren.
2. Schrift **Archivo Black** (SIL Open Font License) nach `schriften/` laden:
   `https://github.com/google/fonts/raw/main/ofl/archivoblack/ArchivoBlack-Regular.ttf`.
3. **Freisteller wählen:**
   - **Mac:** Apples eingebautes Vision. Schreib ein kleines Swift-Programm `maske.swift`, das das Video
     Bild für Bild liest (`AVAssetReader`) und mit `VNGeneratePersonSegmentationRequest` (Qualität
     `balanced`, über einen `VNSequenceRequestHandler`, der glättet über die Zeit) pro Bild eine
     Graustufen-Maske als PNG schreibt (weiß = Person). Aufruf mit `swift maske.swift`, kein Xcode nötig.
     Breite der Masken 540 Pixel reicht, sie werden später hochgerechnet.
   - **Windows:** `mediapipe` in die `.venv` installieren und das Modell „selfie_segmenter“ von Google
     benutzen (`ImageSegmenter`, Ausgabe `category_mask` oder `confidence_masks`), ebenfalls eine
     Maske pro Bild.

### Schritt 1: Video und Titel

- Video im Ordner finden, mit `ffprobe` Größe, Bildrate, Drehung, Farbbereich lesen. Ziel: 1080 × 1920,
  30 fps. Andere Bildraten vorher auf 30 umrechnen, sonst passen Maske und Bild nicht zusammen.
- **Drehung:** Hat das Video eine Drehung in den Metadaten, es vorher mit ffmpeg einmal „gerade“ neu
  ausgeben. Sonst liest das Freistellen auf dem Mac das Bild anders herum als ffmpeg, und die Masken
  passen nicht.
- **Spiegelung:** Ein Standbild ansehen. Steht Schrift im Bild spiegelverkehrt (Frontkamera), Bild und
  Masken waagerecht spiegeln und das der Person in einem Satz sagen, damit sie es abwählen kann. Im Skript
  als Schalter (`--spiegeln` oder `--nicht-spiegeln`), die Entscheidung in `einstellungen.json` im Ordner
  merken, damit das nächste Video sie übernimmt.
- **Titel:** Den Text nimmt die Person aus ihrem Satz („setz ‚HINTER MIR‘ …“). Fehlt er, fragen.
  Großbuchstaben. **Hat der Titel zwei oder mehr Wörter, auf zwei Zeilen teilen**, dann wird die Schrift
  deutlich größer. Höchstens 12 Zeichen pro Zeile, beide Zeilen mittig, Zeilenabstand 0,15 der
  Versalhöhe.
- **Wann:** Standard ab Sekunde 0 für 3 Sekunden. Nennt die Person ein Wort aus dem Video, bei dem der
  Titel kommen soll, finde die Stelle mit einer Abschrift (`faster-whisper`, Modell `small`,
  `word_timestamps=True`, Ton vorher mit ffmpeg als 16-kHz-Zahlenreihe einlesen) und setz den Titel ab
  diesem Wort.

### Schritt 2: Masken rechnen

- Freisteller aus Schritt 0 über das ganze Video laufen lassen, Masken nach `ausgabe/.../maske/`.
- Prüfen: Anzahl der Masken = Anzahl der Bilder. Weicht sie ab, abbrechen und melden.

### Schritt 3: Wo der Titel hinkommt

- Aus den Masken der ersten Sekunde, in der der Titel steht, die **Oberkante des Kopfes** bestimmen:
  oberste Zeile mit Person im mittleren Drittel des Bildes, als Median über diese Masken.
- Den Titel **einmal** festlegen und dann nicht mehr bewegen. Nur die Person bewegt sich davor.
- **Größe:** so groß, dass die sichtbare Schrift der längsten Zeile 88 % der Bildbreite füllt. Gemessen an
  der echten Tinte: jede Zeile einzeln zeichnen und auf die Buchstaben zuschneiden, sonst rechnet die
  Bildbibliothek die Luft neben den Buchstaben mit.
- **Höhe:** Die unterste Zeile wird zu **45 % ihrer Schrifthöhe** vom Kopf verdeckt: Ihre Unterkante liegt
  also 0,45 × Schrifthöhe unter der Kopfoberkante. Schrifthöhe heißt hier die Höhe eines Großbuchstabens
  (gemessen am „H“). Weil der Kopf rund ist, verschwindet so nur ein Viertel bis ein Drittel des Wortes,
  es bleibt lesbar.
- **Platz:** Die Oberkante der Schrift liegt nie höher als 6 % der Bildhöhe. Reicht der Platz über dem Kopf
  dafür nicht, den Titel verkleinern, bis er passt, und der Person sagen, dass beim nächsten Dreh mehr
  Luft über dem Kopf hilft.
- Weiß, kein Rand, kein Schatten: Der Effekt lebt davon, dass die Schrift wie an der Wand wirkt.

### Schritt 4: Zusammensetzen

Pro Bild, in dieser Reihenfolge von hinten nach vorn:

1. das Originalbild,
2. der Titel (nur in seiner Zeit; rein mit 0,1 s Einblenden, raus hart),
3. die Person aus dem Originalbild, ausgeschnitten mit der Maske dieses Bildes.

- Maske auf Bildgröße hochrechnen, Rand weich machen (Weichzeichner mit Radius 2) und **über drei Bilder
  mitteln** (das Bild davor, dieses, das danach), sonst flackert der Rand an den Haaren.
- Bilder mit ffmpeg einlesen und wieder ausgeben (Rohdaten über eine Pipe). Beim Einlesen
  `-fps_mode passthrough` setzen, sonst schiebt ffmpeg ein Bild mehr in die Pipe als es Masken gibt, und
  das Programm hängt. Die Pipe schließen, bevor auf ffmpeg gewartet wird.
- **Farben:** Beim Einlesen Farbbereich und Matrix der Quelle angeben, beim Ausgeben
  `scale=out_range=tv:out_color_matrix=bt709`, sonst werden Hauttöne leicht falsch.
- Kodierung: H.264, `crf 18`, `yuv420p`, Ton aus dem Original kopieren (`-c:a copy`), `-movflags
  +faststart`. Ergebnis: `ausgabe/.../kopf.mp4`.
- Zuerst nur die Sekunden mit Titel plus eine Sekunde danach rechnen, als `ausgabe/.../vorschau.mp4`, und
  der Person zeigen. Erst wenn die passt, das ganze Video.

### Schritt 5: Selbst prüfen

1. Drei Standbilder aus der Titelzeit ziehen (Anfang, Mitte, Ende) und ansehen:
   - Verdeckt der Kopf den unteren Teil der Schrift, aber bleibt das Wort lesbar?
   - Ist die Schrift nicht über dem Gesicht und nicht oben angeschnitten?
   - Sind die Haare sauber, kein farbiger Saum, keine Löcher im Kopf?
2. Ein Standbild **ohne** Titel ansehen: Sieht es aus wie das Original? Mit dem Auge vergleichen. Ein
   Zahlenvergleich in RGB zeigt wegen der Farbumrechnung immer ein, zwei Stufen Unterschied, das ist
   kein Fehler.
3. Erst dann melden: wo das Video liegt, und dass es mit „setz ‚NEUES WORT‘ hinter meinen Kopf“ jederzeit
   wieder geht.

### Wenn die Person nachbessern will

- „Mehr verdeckt“ oder „weniger verdeckt“: die 45 % in Schritten von 10 ändern.
- „Größer“ oder „kleiner“: die 88 % ändern.
- „Andere Farbe“: Farbe des Titels ändern, alles andere bleibt.
- „Später“ oder „länger“: Startwort oder Dauer ändern.
