# Tokyo Trip Companion

Persönliche Reise-Planer-App für eine Japan-Reise (Tokyo-Basis, 05.–25.11.2026, Unterkunft in Ebisu). Läuft als **Progressive Web App (PWA)** – installierbar auf dem Handy-Homescreen, funktioniert (größtenteils) offline.

## Tech-Stack

- **Reines Vanilla HTML/CSS/JavaScript** – kein Build-Step, kein Framework, keine npm-Dependencies
- Alles in einer Datei (`index.html`), Rendering per manuellem `innerHTML`-Update (kein virtuelles DOM)
- Externe Libraries werden per CDN `<script>`-Tag eingebunden (kein `npm install` nötig):
  - **Leaflet** (`leaflet.js`) – Fallback-Kartenengine mit OpenStreetMap-Kacheln
  - **MapTiler SDK JS** (`maptiler-sdk.umd.min.js`) – zweite Kartenengine-Option (Vektor-Karten, optional per API-Key)
  - **Google Maps JavaScript API** – wird dynamisch nachgeladen (nicht im `<head>`, da der Key erst zur Laufzeit bekannt ist), bevorzugte Kartenengine, falls Key gesetzt

## Dateien

| Datei | Zweck |
|---|---|
| `index.html` | Die komplette App (HTML + CSS + JS in einer Datei) |
| `manifest.json` | PWA-Manifest (Name, Icons, Standalone-Modus) fürs Homescreen-Icon |
| `sw.js` | Service Worker – cached den App-Shell für Offline-Nutzung (network-first, cache-fallback) |
| `icon-192.png` / `icon-512.png` | App-Icons fürs Homescreen |

## Deployment

Gehostet auf **GitHub Pages** (statisches Hosting, kostenlos). Einfach die Dateien im Repo überschreiben und pushen – GitHub Pages deployed automatisch neu. Wichtig: Der **Origin/URL-Pfad darf sich nicht ändern**, sonst geht der lokal gespeicherte Reisedaten-Stand des Nutzers verloren (siehe Storage-Abschnitt unten).

## Datenmodell (`state`-Objekt, siehe `defaultState()` in `index.html`)

```js
{
  overview: { hinflug, ankunft, rueckflug, unterkunft, naechte, notizen },
  checklists: {
    einreise: [{id, text, done}],
    packliste: [{id, text, done}]
  },
  categories: [{id, name, color}],       // z.B. Anime, JDM, Geschichte, Ruhige Viertel, Aussicht, Essen — Nutzer kann eigene hinzufügen
  neighborhoods: [{id, name, lat, lng}], // echte GPS-Koordinaten (Tokyo + Tagestrip-Ziele); Nutzer kann eigene per Geocoding (Nominatim) hinzufügen
  places: [{id, name, neighborhoodId, categoryIds:[], notes, done}],
  filters: { cats:[], nb:[] },           // aktive Filter im Orte-Tab
  settings: { maptilerKey:'', googleMapsKey:'' },
  activeTab: 'overview' | 'entry' | 'pack' | 'places'
}
```

Alles im State ist über die UI editierbar (Checklisten-Items, Orte, Kategorien, Stadtteile, Übersicht-Felder) — der Nutzer wollte explizit eine voll anpassbare App, keine hartkodierten Inhalte.

## Storage

`storageAdapter` (in `index.html`) abstrahiert die Persistenz:
- Nutzt `window.storage` (Claude-Artifact-API), **falls verfügbar** (z.B. wenn die Datei noch als Claude-Artefakt läuft)
- **Fallback auf `localStorage`**, wenn `window.storage` nicht existiert (Normalfall bei GitHub-Pages-Hosting / lokal geöffneter Datei)
- Ein Key: `trip-state`, Wert = `JSON.stringify(state)`
- Debounced Save (250ms) bei jeder Änderung über `mutate(fn)`

⚠️ **Kein echter Dateisystem-Zugriff.** Mobile Browser erlauben keinen Zugriff auf einen selbstgewählten lokalen Ordner — das war ein expliziter Wunsch des Nutzers, ist aber technisch nicht möglich. `localStorage` ist die bestmögliche Alternative (persistiert dauerhaft auf dem Gerät, gebunden an die exakte URL/Origin).

Es gibt außerdem eine **manuelle Export/Import-Sicherung** (Reise-Tab, unten): zeigt den kompletten State als JSON-Text zum Kopieren/Einfügen, als Backup unabhängig von Code-Änderungen.

## Karten-Engine (Orte-Tab)

Drei-Stufen-Fallback, implementiert in `initMap()`:

1. **Google Maps** (bevorzugt, falls `settings.googleMapsKey` gesetzt) — beste Label-Qualität (zeigt automatisch übersetzte + lokale Namen), aber Nutzer braucht eigenen API-Key (Google-Cloud-Konto + hinterlegte Kreditkarte, real. kostenlos bei diesem Nutzungsvolumen: <10.000 Kartenaufrufe/Monat)
2. **MapTiler** (falls `settings.maptilerKey` gesetzt, kein Google-Key) — kostenlos ohne Kreditkarte, nutzt `applyBilingualLabels()` um zweizeilige Labels (romanisiert + Original) selbst in die Kartenstil-Layer einzubauen, da MapTilers Sprachumschaltung allein zu lückenhaft war
3. **Leaflet + OpenStreetMap** (Fallback ohne jeden Key) — komplett kostenlos, aber Kachel-Beschriftung bleibt auf Japanisch

Marker/Filter-Logik (`markerStyle()`) ist Engine-unabhängig; jede der drei `init*Map()`-Funktionen implementiert nur das Rendering für ihre jeweilige Library.

Google-Maps-Marker zeigen Namen erst ab Zoomstufe `GOOGLE_LABEL_ZOOM_THRESHOLD` (aktuell 13), um die Karte bei weiter Ansicht nicht zu überladen.

## Design

Dunkles Farbschema, Akzentfarbe **Lila** (`--accent: #a78bfa`), Typografie: Oswald (Headlines/Transit-Sign-Look), Inter (Body), JetBrains Mono (Daten/Zahlen). Signature-Element war ursprünglich eine stilisierte Yamanote-Linien-Karte (SVG), wurde aber durch echte interaktive Karten (Leaflet/MapTiler/Google) ersetzt, da der Nutzer eine "echte", zoombare Karte wollte.

## Offene Punkte / mögliche nächste Schritte

- Icons sind einfache generierte Platzhalter (lila Kreis + Pin-Symbol) — könnten durch was Individuelleres ersetzt werden
- Kein automatisiertes Test-Setup (kleines Vanilla-JS-Projekt, manuelles Testen im Browser)
- `applyBilingualLabels()` ist Best-Effort (hängt von Datenverfügbarkeit in den MapTiler/OSM-Kartendaten ab) — bei Google Maps nicht nötig, da Google das selbst übernimmt
