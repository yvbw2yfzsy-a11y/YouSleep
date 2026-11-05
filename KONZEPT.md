# YouSleep – Konzept

Eigene YouTube-Oberfläche für iPhone-Safari. Läuft auf dem Raspberry Pi unter `http://<pi-ip>/ys`.
Ohne Shorts, ohne Werbung, dauerhaft angemeldet, mit Startseite, Abobox und schnellem Umschalter Deutsch/Englisch.

Status: Konzept, noch kein Code.

## Kennzeichnung

YouTube ändert seine Technik ständig. Deshalb steht bei technischen Aussagen, wie sicher sie sind:

- **[gesichert]**: dokumentiert oder breit bekannt, Stand meines Wissens (bis Mitte 2026).
- **[Vermutung]**: plausibel, aber von mir nicht geprüft.
- **[unklar]**: weiß ich nicht, oder kann sich inzwischen geändert haben. Muss im Prototyp getestet werden.

---

## 1. Kurzfassung

Ein normaler Chromium läuft unsichtbar auf dem Pi. Er ist bei Google angemeldet und bleibt es.
Eine kleine eigene Erweiterung in diesem Chromium holt die Daten von YouTube: Startseite, Abobox, Suche, Videoinfos – mit deinem Login.
Der Pi filtert daraus Shorts und Werbung. Er gibt nur saubere Daten an eine eigene Web-App weiter, die auf dem iPhone aussieht wie die YouTube-App.
Videos spielt Safari mit seinem eigenen Player ab. Der Videostream kommt über yt-dlp, nicht über YouTubes Player. Deshalb wird Videowerbung gar nicht erst geladen.

---

## 2. Wo ich dir widerspreche

### 2.1 Den Chromium-Bildschirm aufs iPhone zu streamen ist der falsche Weg

Falls du gemeint hast, dass das iPhone das Bild des Chromium sieht (wie bei Fernwartung): Davon rate ich ab.

- Das fühlt sich nie wie eine App an. Scrollen, Tippen und Tastatur laufen über eine Bildübertragung, mit spürbarer Verzögerung.
- Das Video müsste auf dem Pi abgespielt, abgefilmt und neu codiert werden. Der Raspberry Pi 5 hat keinen Hardware-Encoder für H.264 **[gesichert]**. Software-Encoding in Echtzeit bei 1080p schafft er nicht flüssig **[Vermutung, sehr wahrscheinlich]**. Der Pi 4 hat einen H.264-Encoder **[gesichert]**, trotzdem wäre es ruckelig und mit Ton-Versatz.
- AirPlay, Bild-in-Bild und Steuerung im Sperrbildschirm gingen damit nicht.

Stattdessen: Chromium bleibt die „Hand, die in YouTube reingreift“. Er empfängt die Daten mit deinem Login. Weitergegeben werden aber **Daten**, nicht Bilder.
Das ist deine Idee „was der Chromium empfängt, gibt der Pi weiter“ – nur auf Datenebene.

### 2.2 Die tägliche Adblock-Liste ist nicht der kritische Teil

In diesem Aufbau läuft kein YouTube-Player. Werbevideos werden also gar nicht erst geladen.
Werbekarten in den Feeds entfernt eine **Whitelist**: Nur bekannte Video-Elemente werden angezeigt, alles andere fliegt raus.
Eine klassische Filterliste wie bei uBlock braucht man dafür nicht.

Was wirklich jeden Tag aktuell sein muss, ist **yt-dlp**. YouTube ändert regelmäßig seine Schutzmechanismen. Ohne Update spielen Videos irgendwann nicht mehr ab **[gesichert]**.
Der 3-Uhr-Job aktualisiert deshalb vor allem yt-dlp, dazu unsere eigene Filterregel-Datei (Abschnitt 8).

### 2.3 uBlock Origin läuft im Chromium nicht mehr wie früher

Google hat Erweiterungen nach Manifest V2 abgeschaltet. Das klassische uBlock Origin läuft in Chrome nicht mehr **[gesichert für Google Chrome; für das Chromium von Raspberry Pi OS Vermutung: gleich, weil es Upstream folgt]**.
Es gibt nur noch uBlock Origin Lite. Dessen Listen kommen mit Updates der Erweiterung, nicht als tägliche Downloads **[gesichert, so funktioniert declarativeNetRequest]**.

Für unseren Aufbau ist das egal. Der Chromium zeigt niemandem Werbung, er ist nur Datenquelle. uBlock Origin Lite ist optional, nur um Rechenlast zu sparen.

### 2.4 „Nur deutsche“ oder „nur englische“ Startseite gibt es bei YouTube nicht

YouTube-Anfragen kennen Sprache (`hl`) und Region (`gl`) **[gesichert]**.
Das ändert sicher die Oberflächensprache und teilweise die Empfehlungen **[Vermutung – wie stark, weiß ich nicht]**.
Einen harten Filter „nur Videos auf Deutsch“ bietet YouTube nicht. Deshalb zwei Stufen, siehe Abschnitt 7.

### 2.5 Verlauf und Empfehlungen leiden

Wenn Videos in unserem eigenen Player laufen, weiß YouTube nicht, dass du sie gesehen hast.
Folge: Der YouTube-Verlauf bleibt leer, und die Startseite wird mit der Zeit schlechter personalisiert **[Vermutung, aber naheliegend]**.
Lösungsidee in Abschnitt 6.3. Das ist eine der unsichersten Stellen im Konzept.

### 2.6 Risiko für das Google-Konto

YouTubes Nutzungsbedingungen verbieten automatisierten Zugriff und das Umgehen von Werbung **[gesichert]**.
Was das konkret für ein Konto bedeutet, weiß ich nicht **[unklar]**.
Bekannt ist: YouTube blockt Programme wie yt-dlp regelmäßig **[gesichert]**. Die yt-dlp-Doku warnt, dass Nutzung mit Konto-Cookies zu Sperren des Kontos führen kann **[gesichert, Stand 2025]**.

Darum holt das System Videostreams standardmäßig **ohne** Konto. Das Konto wird nur für Feeds und Abos verwendet (Abschnitt 6.2).
Nur privat nutzen, nicht veröffentlichen.

### 2.7 Ohne HTTPS ist es nur „fast“ eine App

„Zum Home-Bildschirm“ und Start im Vollbild klappen auch über `http://` **[Vermutung]**.
Einen Service Worker (schneller Start, App-Hülle offline) erlaubt Safari aber nur über HTTPS **[gesichert]**.
Für den Anfang reicht HTTP im Heimnetz. HTTPS kommt in Phase 3 (Abschnitt 13).

---

## 3. Architektur

```
iPhone (Safari / Home-Bildschirm-App)
   │  http://<pi-ip>/ys
   ▼
nginx (bestehend, craft.conf)
   ├─ /ys/        → statische Web-App      (/var/www/craft/ys)
   ├─ /ys/api/    → ys-core   127.0.0.1:8090
   └─ /ys/setup/  → noVNC     127.0.0.1:6080   (nur zum Anmelden)
                        │
   ┌────────────────────┴──────────────────────────┐
   │ ys-core (Python, FastAPI)                     │
   │  ├─ Feed-Dienst   → fragt über die Bridge an  │
   │  ├─ Filter        → Shorts + Werbung raus     │
   │  ├─ Player-Dienst → yt-dlp (+ Deno) → Streams │
   │  ├─ Remux         → ffmpeg, nur als Fallback  │
   │  └─ Cache         → SQLite                    │
   └────────────────────┬──────────────────────────┘
                        │ WebSocket 127.0.0.1:8090/bridge
   ┌────────────────────┴──────────────────────────┐
   │ Chromium (normal gestartet, nicht ferngesteuert)
   │  auf virtuellem Bildschirm (Xvfb :99)         │
   │  Profil: /var/lib/yousleep/chromium           │
   │  ├─ bei Google angemeldet (dauerhaft)         │
   │  ├─ ein youtube.com-Tab, immer offen          │
   │  └─ Erweiterung „ys-bridge“                   │
   └────────────────────┬──────────────────────────┘
                        ▼
                youtube.com (InnerTube-API)
```

Aufgabe der Teile:

- **Chromium + ys-bridge**: Login-Tresor und Datenquelle.
- **ys-core**: übersetzt und filtert. Einzige Stelle mit Logik.
- **Web-App**: reine Oberfläche. Spricht nur mit dem Pi, außer beim Laden von Vorschaubildern.
- **nginx**: wie bisher, plus drei `location`-Blöcke.

### Beispiel: Startseite öffnen

1. iPhone öffnet `/ys/`. nginx liefert die Web-App aus.
2. Web-App fragt `/ys/api/feed/home?lang=de`.
3. ys-core prüft den Cache (gültig 10 Minuten). Bei Treffer weiter mit Schritt 6.
4. ys-core schickt über die Bridge: `browse`, `browseId: FEwhat_to_watch`, `hl=de`, `gl=DE`.
5. Die Erweiterung führt die Anfrage im youtube.com-Tab aus, mit deinem Login. Die Rohdaten gehen zurück an ys-core.
6. ys-core filtert und liefert einfache Daten: `{items: [{id, title, channel, thumb, duration, views, published}], continuation}`.
7. Web-App zeigt die Karten. Beim Runterscrollen lädt sie mit `continuation` nach.

---

## 4. Chromium und Erweiterung „ys-bridge“

### 4.1 Warum Erweiterung statt Fernsteuerung

Man könnte Chromium auch per DevTools-Protokoll oder Playwright fernsteuern.
Google blockt die Anmeldung in ferngesteuerten Browsern aber häufig („Dieser Browser oder diese App ist möglicherweise nicht sicher“) **[gesichert, bekanntes Problem]**.
Ein normal gestarteter Chromium mit Erweiterung ist von außen ein gewöhnlicher Browser **[Vermutung]**.
Deine Idee mit der Erweiterung ist hier also der bessere Weg.

### 4.2 Aufbau (Manifest V3)

- **Service Worker** der Erweiterung: hält eine WebSocket-Verbindung zu `ws://127.0.0.1:8090/bridge`. Seit Chrome 116 hält WebSocket-Verkehr den Service Worker am Leben **[gesichert]**.
- **Content Script, zweiteilig**, im youtube.com-Tab:
  - Teil „isolated“ nimmt Aufträge vom Service Worker an und reicht sie per `window.postMessage` weiter.
  - Teil „main“ (`world: MAIN`) führt die Anfrage aus: `POST https://www.youtube.com/youtubei/v1/<endpunkt>`. Er nutzt die Konfiguration, die YouTube selbst in die Seite lädt (`ytcfg`: API-Key, Client-Version, Kontext), und den Login der Seite (Cookies + `SAPISIDHASH`-Header) **[gesichert, so arbeitet YouTubes eigene Web-App]**.
- Ablauf: ys-core schickt `{id, endpoint, body, hl, gl}` → Service Worker → Content Script → YouTube → Rohdaten zurück.
- Die Erweiterung filtert nichts. Alle Logik liegt in ys-core. So muss man bei YouTube-Änderungen nur eine Stelle anpassen.

### 4.3 Benutzte InnerTube-Endpunkte

Diese Endpunkte nutzen auch Invidious, NewPipe und yt-dlp **[gesichert, Namen Stand meines Wissens]**.

| Zweck | Endpunkt | Parameter |
|---|---|---|
| Startseite | `browse` | `browseId: FEwhat_to_watch` |
| Abobox | `browse` | `browseId: FEsubscriptions` |
| Liste der abonnierten Kanäle | `browse` | `browseId: FEchannels` |
| Verlauf | `browse` | `browseId: FEhistory` |
| Später ansehen | `browse` | `browseId: VLWL` |
| Kanalseite | `browse` | Kanal-ID + Tab-Parameter |
| Videoseite (Titel, Beschreibung, ähnliche Videos) | `next` | `videoId` |
| Suche | `search` | `query` |
| Nachladen (Endlos-Scroll) | jeweiliger Endpunkt | `continuation` |
| Phase 2: Abo, Mag ich, Später ansehen | `subscription/subscribe`, `like/like`, `browse/edit_playlist` | – |

### 4.4 Start und Betrieb

- Chromium läuft als systemd-Dienst auf dem virtuellen Bildschirm Xvfb.
- Wichtige Optionen: eigenes Profilverzeichnis, `--password-store=basic` (sonst sucht Chromium einen Schlüsselbund, den es ohne Desktop nicht gibt) **[gesichert]**, `--no-first-run`.
- Erweiterung laden: per `--load-extension=/opt/yousleep/extension` oder einmalig im Entwicklermodus unter `chrome://extensions` über noVNC.
  Google Chrome unterstützt `--load-extension` seit Version 137 nicht mehr **[gesichert]**. Ob das Pi-Chromium es noch kann, weiß ich nicht **[unklar]**. Der Entwicklermodus-Weg bleibt im Profil gespeichert **[Vermutung]**.
- Ein Wächter in ys-core startet Chromium neu, wenn die Bridge 60 Sekunden nicht antwortet.

---

## 5. Filter: keine Shorts, keine Werbung

### 5.1 Prinzip: Whitelist

ys-core wandelt jede YouTube-Antwort in ein eigenes, einfaches Format um.
Dabei übernimmt es **nur bekannte Element-Typen**. Alles Unbekannte wird verworfen und protokolliert.
Neue Werbeformate fallen dadurch automatisch raus, auch wenn YouTube sie neu erfindet.

Nachteil: Ein neues, normales Videoformat fiele auch raus. Das meldet der nächtliche Selbsttest (Abschnitt 8), dann wird die Regel ergänzt.

Die Regeln liegen in `filters/rules.json` im Repo, nicht im Code. Das ist die „Filterliste“ dieses Systems. Der Nachtjob aktualisiert sie.

### 5.2 Erlaubte Elemente (Startwert)

`richItemRenderer` mit `videoRenderer` oder `lockupViewModel` vom Typ Video, `videoRenderer`, `compactVideoRenderer`, `gridVideoRenderer`, `playlistRenderer`, `channelRenderer`, `continuationItemRenderer`, `feedFilterChipBarRenderer` (Themen-Chips).

Element-Namen: **[Stand meines Wissens; YouTube benennt sie öfter um, aktueller Stand unklar]**.

### 5.3 Shorts

- Verwerfen: `reelShelfRenderer`, `reelItemRenderer`, `shortsLockupViewModel`, Regale (`richShelfRenderer`) mit Shorts-Kennzeichen.
- Verwerfen: alles, dessen Link ein `reelWatchEndpoint` ist oder `/shorts/` enthält. Das greift auch, wenn YouTube die Element-Namen ändert.
- Kanalseiten: Tab „Shorts“ wird nicht angezeigt.
- Suche: Shorts-Regale fallen raus.
- Geteilte `/shorts/`-Links: werden gesperrt (offene Frage 3).
- Optionaler Zusatzfilter „Videos bis 3 Minuten ausblenden“: standardmäßig **aus**, weil er auch normale kurze Videos trifft. Shorts dürfen seit Oktober 2024 bis 3 Minuten lang sein **[gesichert]**.

### 5.4 Werbung

- **In Feeds und Suche**: Werbekarten stehen nicht in der Whitelist, fallen also weg. Bekannte Typen zur Kontrolle im Log: `adSlotRenderer`, `promotedSparklesWebRenderer`, `promotedVideoRenderer`, `bannerPromoRenderer`, `statementBannerRenderer`, `searchPyvRenderer` **[Stand meines Wissens]**.
- **Vor und in Videos**: Unser Player spielt nur den Videostream. Werbespots sind bei YouTube eigene Videos, die YouTubes Player zusätzlich lädt **[gesichert für den bisherigen Stand]**.
  YouTube hat aber Werbung getestet, die direkt in den Videostream eingebaut ist **[gesichert, dass es Tests gab]**. Ob das die Streams betrifft, die yt-dlp holt, weiß ich nicht **[unklar]**. Falls ja, ließe sich das kaum herausfiltern.
- **Eingesprochene Werbung der YouTuber** („Dieses Video wird gesponsert von …“): optional über die öffentliche **SponsorBlock**-API automatisch überspringen **[gesichert, API frei verfügbar]**. Standard an oder aus: offene Frage 4.

---

## 6. Login und Wiedergabe

### 6.1 Einmal anmelden, dauerhaft angemeldet

1. In der Web-App unter „Du → Konto einrichten“ öffnet sich `/ys/setup/`.
2. Dort siehst du per noVNC den echten Chromium auf dem Pi.
3. Du meldest dich bei Google an, mit Zwei-Faktor.
4. Das Profil bleibt dauerhaft auf dem Pi. Das iPhone braucht danach keine Anmeldung mehr, es spricht nur mit dem Pi.

Damit es so bleibt:

- Der youtube.com-Tab lädt alle paar Stunden neu. Google erneuert dabei die Sitzungs-Cookies wie bei einem normal genutzten Browser.
  **[Vermutung]**, dass das eine Abmeldung verhindert. Google kann trotzdem jederzeit eine neue Anmeldung verlangen, etwa nach einer Passwortänderung **[gesichert]**.
- ys-core prüft bei jeder Antwort, ob YouTube „abgemeldet“ meldet. Das genaue Feld muss im Prototyp ermittelt werden **[unklar]**. Wenn abgemeldet: Banner „Neu anmelden“ in der App.
- noVNC läuft nur während der Einrichtung (Abschnitt 12).

Die Anmeldung über noVNC auf dem iPhone ist etwas fummelig. Alternative: einmal Bildschirm und Tastatur an den Pi.

### 6.2 Videos abspielen

Ablauf beim Tippen auf ein Video:

1. Web-App fragt `/ys/api/video/<id>`.
2. ys-core holt parallel:
   - Videoinfos und ähnliche Videos über die Bridge (`next`, gefiltert),
   - die Stream-Adressen über **yt-dlp** (als Python-Bibliothek im selben Prozess).
3. Die Web-App spielt mit dem eingebauten Safari-Player (`<video playsinline>`) ab. Dadurch gehen Vollbild, Bild-in-Bild und AirPlay **[gesichert für Safari-Video]**.

Formatwahl fürs iPhone, in dieser Reihenfolge:

1. **HLS** (`.m3u8`): spielt Safari nativ, mit automatischer Qualitätsanpassung. YouTube liefert HLS an manche Clients, zum Beispiel den iOS-Client **[gesichert]**. Ob yt-dlp das aktuell ohne Zusatz (PO-Token) bekommt, weiß ich nicht **[unklar]**.
2. **Remux auf dem Pi**: ffmpeg holt getrennt H.264-Video und AAC-Audio und packt beides ohne Neucodierung in HLS um (`-c copy`). Das kostet kaum Rechenleistung **[gesichert]**, auch 1080p ist machbar. H.264 statt VP9/AV1, weil jedes iPhone H.264 in Hardware decodiert **[gesichert]**.
3. **MP4 in 360p** (itag 18): letzte Notlösung. Höhere fertige MP4-Formate bietet YouTube kaum noch an **[gesichert, Stand 2025]**.

Weitere Punkte:

- Stream-Adressen laufen nach etwa 6 Stunden ab **[gesichert]**. Cache daher maximal 5 Stunden.
- Stream-Adressen sind an die IP gebunden, die sie angefragt hat **[gesichert]**. Zuhause haben Pi und iPhone dieselbe öffentliche IP, direktes Laden sollte gehen **[Vermutung]**. Falls nicht, oder unterwegs: Stream läuft über den Pi.
- yt-dlp braucht für YouTube inzwischen eine JavaScript-Laufzeit (Deno) **[gesichert, Stand Ende 2025]**. Teilweise sind „PO-Tokens“ nötig, dafür gibt es Zusatz-Plugins **[gesichert, dass es sie gibt; was im November 2026 gilt: unklar]**.
- Streams werden **ohne** Konto-Cookies geholt (Schutz des Kontos, Abschnitt 2.6). Nur bei altersbeschränkten Videos nutzt yt-dlp die Cookies aus dem Chromium-Profil.

### 6.3 Gesehen-Status an YouTube zurückmelden (Phase 2)

Ziel: Verlauf und Empfehlungen sollen weiter funktionieren.

- Die Antwort des `player`-Endpunkts enthält Tracking-Adressen (`playbackTracking`). YouTubes eigener Player ruft sie beim Abspielen auf **[gesichert, dass es sie gibt]**.
- ys-core könnte sie über die Bridge mit deinem Login aufrufen.
- Ob YouTube das als echtes Ansehen zählt, weiß ich nicht **[unklar]**.
- Plan B: Chromium öffnet das Video stumm in einem Hintergrund-Tab. Das kostet aber deutlich Rechenleistung.

---

## 7. Startseite: Deutsch / Englisch umschalten

**Bedienung**: Ganz links in der Chip-Leiste der Startseite sitzt ein Umschalter **DE | EN**. Ein Tipp wechselt sofort. Die App merkt sich die Wahl.

**Technik, zwei Stufen:**

1. **Sprache und Region an YouTube mitgeben.** DE = `hl=de, gl=DE`, EN = `hl=en, gl=US`. Gilt auch für die Suche. Jede Sprache hat ihren eigenen Cache, der Wechsel ist deshalb sofort da.
2. **Optionaler harter Filter.** ys-core erkennt die Sprache jedes Videotitels mit einem kleinen Modell, z. B. fastText `lid.176.ftz` (ca. 1 MB) **[gesichert]**. Videos der anderen Sprache werden ausgeblendet. ys-core lädt automatisch nach, bis genug Videos da sind.
   Schwäche: Kurze Titel oder Titel wie „VLOG #42“ werden oft falsch erkannt **[gesichert, typische Schwäche]**.

Ob Stufe 1 allein reicht, zeigt erst ein Test mit deinem Konto **[unklar]**.
Die Abobox wird standardmäßig nicht nach Sprache gefiltert. Der Filter lässt sich dort aber zuschalten.

---

## 8. Nächtlicher Job um 03:00

Umsetzung als systemd-Timer: `OnCalendar=*-*-* 03:00:00` mit `Persistent=true`. So wird ein verpasster Lauf nachgeholt, falls der Pi um 3 Uhr aus war **[gesichert]**.
crontab geht auch, holt verpasste Läufe aber nicht nach.

Schritte von `ys-update`:

1. **yt-dlp** im eigenen Python-venv aktualisieren, inklusive Zusatzpaket für die JavaScript-Lösung. **Deno** aktualisieren.
2. PO-Token-Plugin aktualisieren, falls verwendet.
3. **Filterregeln** (`filters/rules.json`) aus dem Repo holen (`git pull`). Das ist die „Adblocker-Liste“ dieses Systems.
4. **Chromium neu starten** (gibt Speicher frei). Warten, bis die Bridge wieder verbunden ist.
5. **Selbsttest:**
   - Noch angemeldet?
   - Laden Startseite und Abobox?
   - Null Shorts im Ergebnis? Geprüft auch unabhängig von den Filterregeln, z. B. `/shorts/` im Rohtext.
   - Neue, unbekannte Element-Typen aufgetaucht? Liste ins Log.
   - Stream-Adresse für ein Testvideo ermittelbar?
6. Ergebnis nach `/ys/api/health`. In der App zeigt ein Punkt am Profilbild Grün oder Rot.
7. Cache leeren. Protokoll im Journal: `journalctl -u ys-update.service`.

Ehrlich: Kein Job garantiert, dass „immer“ alles ohne Werbung und Shorts bleibt. YouTube ändert seine Interna oft **[gesichert]**.
Der Selbsttest sorgt dafür, dass du einen Fehler morgens am roten Punkt siehst, nicht erst beim Schauen.

Dein vorhandenes `craft-update` kenne ich nicht. `ys-update` ist bewusst getrennt, damit ein Fehler im einen Job den anderen nicht blockiert.

---

## 9. Oberfläche (wie die YouTube-App)

### 9.1 Startseite

```
┌────────────────────────────────────┐
│ [Logo] YouSleep          (Q)  (●)  │  Lupe, Profilbild mit Statuspunkt
│ [DE|EN] [Alle] [Musik] [Gaming] …  │  Chip-Leiste, DE/EN ganz links
├────────────────────────────────────┤
│ ┌────────────────────────────────┐ │
│ │                                │ │  Vorschaubild 16:9, volle Breite
│ │                          12:34 │ │  Dauer unten rechts
│ └────────────────────────────────┘ │
│ (o) Titel des Videos, höchstens    │  runder Kanal-Avatar
│     zwei Zeilen …                  │
│     Kanal · 120.000 Aufrufe · vor 2 Tagen
│                                    │
│ ┌────────────────────────────────┐ │
│ │  …                             │ │
├────────────────────────────────────┤
│   Start        Abos        Du      │  Tab-Leiste: kein Shorts, kein „+“
└────────────────────────────────────┘
```

### 9.2 Bereiche

- **Start**: Chip-Leiste (DE|EN und YouTubes Themen-Chips), Videokarten, Endlos-Scroll, Pull-to-Refresh.
- **Abos (Abobox)**: oben eine waagrechte Leiste mit Kanal-Avataren wie in der App, darunter die neuesten Videos chronologisch. Chips: Alle, Heute, Live.
- **Du**: Verlauf, Später ansehen, Playlists, „Mag ich“-Videos, Einstellungen (Sprache, 3-Minuten-Filter, SponsorBlock, neu anmelden, Systemstatus).
- **Suche**: Lupe oben rechts, Vorschläge, Ergebnisse gefiltert.
- **Videoseite**: Player oben fest, Titel, Kanalzeile mit Abo-Knopf, Aktionsleiste (Mag ich, Teilen, Speichern), aufklappbare Beschreibung, ähnliche Videos (gefiltert), Kommentare ab Phase 3.
- Nicht vorhanden: Shorts-Tab, „+“-Knopf, Shorts-Regale, Werbekarten.

### 9.3 Aussehen

- Dunkles Design als Standard, Farben angelehnt an YouTube: Hintergrund `#0f0f0f`, Flächen `#272727`, Text `#f1f1f1`, Nebentext `#aaaaaa`. Helles Design folgt der iOS-Einstellung.
- Systemschrift (`-apple-system`).
- Platzhalter-Karten (Skeleton) beim Laden, damit es sich schnell anfühlt.
- Eigener Name und eigenes Icon („YouSleep“), **nicht** das YouTube-Logo. Layout und Farben nachzuempfinden ist für private Nutzung unproblematisch **[Vermutung, keine Rechtsberatung]**. Logo und Marke zu kopieren wäre es nicht.

### 9.4 Damit es sich unter iOS wie eine App anfühlt

- `manifest.webmanifest` mit `display: standalone`, `start_url: /ys/`, `scope: /ys/`.
- `apple-mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style: black-translucent`, `apple-touch-icon` in 180×180 **[gesichert]**.
- `viewport-fit=cover` und `env(safe-area-inset-*)` für Notch und Home-Leiste **[gesichert]**.
- Single-Page-App ohne Neuladen beim Navigieren, Wischgeste zurück.
- Vorschaubilder lädt Safari direkt von `i.ytimg.com`, ohne Login **[gesichert]**. Das entlastet den Pi.
- Home-Bildschirm-Apps haben unter iOS einen eigenen Speicher, getrennt von Safari **[gesichert]**. Egal, weil der Login auf dem Pi liegt. Lokal liegen nur Einstellungen wie DE/EN.
- Ob Video im Hintergrund oder bei gesperrtem Bildschirm weiterläuft, ist bei Home-Bildschirm-Apps unsicher **[unklar, iOS-Verhalten hat sich mehrfach geändert]**.
- **Feste IP für den Pi** im Router einstellen (DHCP-Reservierung). Sonst zeigt das Icon irgendwann ins Leere.

---

## 10. Einbindung in dein bestehendes Setup

Dein nginx hat in `/etc/nginx/conf.d/craft.conf` einen `server`-Block mit `root /var/www/craft` und `autoindex on`. Das passt gut:

- Die gebaute Web-App liegt unter `/var/www/craft/ys/`. Sie erscheint damit automatisch in deiner App-Liste.
- In den `server`-Block kommt eine Zeile `include /etc/nginx/snippets/yousleep.conf;`.
- `gzip_types` wird um `application/json` ergänzt (API-Antworten).

Inhalt von `yousleep.conf`, sinngemäß (Skizze, nicht getestet):

```nginx
location /ys/ {
    try_files $uri $uri/ /ys/index.html;        # Single-Page-App
}

location /ys/api/ {
    allow 192.168.0.0/16;                       # nur Heimnetz, an dein Netz anpassen
    deny  all;
    proxy_pass http://127.0.0.1:8090/api/;
    proxy_http_version 1.1;
    proxy_buffering off;                        # für Video über den Pi
    proxy_read_timeout 1h;
}

location /ys/setup/ {                           # noVNC, nur zum Anmelden
    allow 192.168.0.0/16;
    deny  all;
    auth_basic "YouSleep Setup";
    auth_basic_user_file /etc/nginx/yousleep.htpasswd;
    proxy_pass http://127.0.0.1:6080/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

### Zusätzliche Pakete

Debian-Pakete: `xvfb`, `x11vnc`, `novnc`, `websockify`, `ffmpeg`, `python3-venv`, `git`, Chromium (Paketname je nach Raspberry-Pi-OS-Version `chromium` oder `chromium-browser`).
Dazu Deno über den offiziellen Installer.

### Dienste (systemd)

| Dienst | Aufgabe |
|---|---|
| `ys-xvfb.service` | virtueller Bildschirm `:99` |
| `ys-chromium.service` | Chromium mit Profil und ys-bridge |
| `ys-core.service` | Backend auf `127.0.0.1:8090` |
| `ys-novnc.service` | noVNC auf `127.0.0.1:6080`, nur bei Bedarf gestartet |
| `ys-update.service` + `ys-update.timer` | Nachtjob 03:00 |

Alle Dienste laufen unter einem eigenen Systembenutzer `yousleep`, nicht als root.

### Verzeichnisse

| Pfad | Inhalt |
|---|---|
| `/opt/yousleep/` | Code (dieses Repo), Python-venv, Erweiterung |
| `/var/lib/yousleep/chromium/` | Chromium-Profil – das ist dein Login |
| `/var/lib/yousleep/cache.db` | SQLite-Cache |
| `/var/www/craft/ys/` | gebaute Web-App |

### Internes API (Entwurf)

| Methode | Pfad | Inhalt |
|---|---|---|
| GET | `/api/feed/home?lang=de&cont=…` | Startseite, gefiltert |
| GET | `/api/feed/subscriptions?cont=…` | Abobox |
| GET | `/api/channels` | abonnierte Kanäle |
| GET | `/api/channel/<id>?tab=videos\|live\|playlists` | Kanalseite, ohne Shorts-Tab |
| GET | `/api/video/<id>` | Infos, ähnliche Videos, Stream-Adresse |
| GET | `/api/stream/<id>/master.m3u8` | Remux-Fallback über den Pi |
| GET | `/api/search?q=…&lang=de` | Suche |
| GET | `/api/health` | Status aus dem Nachtjob |
| POST | `/api/action/<like\|subscribe\|watchlater>` | Phase 2 |
| WS | `/bridge` | nur intern für die Erweiterung, nicht über nginx erreichbar |

---

## 11. Hardware

- Minimum Raspberry Pi 4 mit 4 GB, empfohlen Pi 5 mit 4–8 GB **[Vermutung, auf Basis typischer Chromium-Last]**.
- Speicher grob: Chromium mit einem Tab 0,5–1 GB, ys-core ca. 150 MB, yt-dlp mit Deno kurzzeitig ca. 200 MB **[Vermutung, muss gemessen werden]**.
- SSD statt SD-Karte empfohlen. Chromium schreibt ständig ins Profil, SD-Karten verschleißen dabei.
- Pi per LAN-Kabel anschließen, falls Videos über den Pi laufen.

---

## 12. Sicherheit

- Auf dem Pi liegt eine **voll angemeldete Google-Sitzung**. Wer `/ys/setup/` erreicht, kann im Browser dein Google-Konto bedienen, also auch Gmail und Kontoeinstellungen. Deshalb: Passwort davor, nur aus dem Heimnetz, noVNC nach der Anmeldung abschalten.
- `/ys/api/` nur aus dem Heimnetz erreichbar. Optional eine PIN in der App.
- **Keine Portfreigabe** im Router. Für unterwegs: Tailscale oder WireGuard (Phase 3).
- Bridge und Backend lauschen nur auf `127.0.0.1`.
- Optional: ein eigenes Google-Konto nur für YouSleep, um das Hauptkonto zu schützen. Nachteil: Die Abos müssten dort neu angelegt werden.

---

## 13. Umsetzungsphasen

**Phase 0 – Machbarkeit (zuerst, weil hier das Konzept scheitern kann)**

- Chromium auf Xvfb, Anmeldung über noVNC. Klappt die Google-Anmeldung? Hält sie über mehrere Tage?
- yt-dlp mit Deno auf dem Pi: Gibt es HLS oder H.264-Streams? Wie lange dauert das Ermitteln? Ziel: unter 3 Sekunden.
- Bridge-Prototyp: liefert sie Startseite und Abobox als Rohdaten?

**Phase 1 – Grundversion**

Startseite, Abobox, Videoseite mit Wiedergabe, Shorts- und Werbefilter, DE/EN-Umschalter, Home-Bildschirm-Icon, Nachtjob mit Selbsttest.

**Phase 2**

Suche, Kanalseiten, Verlauf, Später ansehen, Playlists, Knöpfe für Abo und Mag ich, Gesehen-Status zurückmelden, SponsorBlock.

**Phase 3**

HTTPS (eigene lokale Zertifizierungsstelle mit Profil aufs iPhone, oder Tailscale-Zertifikat), Service Worker, Zugriff von unterwegs, Mini-Player, Kommentare.

---

## 14. Offene Fragen

1. Welches Pi-Modell mit wie viel RAM? SD-Karte oder SSD?
2. Hast du mit „was der Chromium empfängt, gibst du weiter“ Daten gemeint (so geplant) oder wirklich das Bild?
3. Geteilte Shorts-Links: sperren oder als normales Video abspielen?
4. SponsorBlock standardmäßig an?
5. Nur zuhause oder auch unterwegs?
6. Soll der Gesehen-Status an YouTube zurückgemeldet werden? Vorteil: bessere Startseite. Nachteil: mehr Aufwand, Erfolg unsicher.
7. Was macht dein `craft-update`? Soll `ys-update` dort mit hinein?
