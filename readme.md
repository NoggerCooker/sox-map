# Analyse des Projekts `sox-map`

## 1. Zweck

`sox-map.html` ist eine eigenständige Browser-Anwendung zur Darstellung von Straßenlampen aus OpenStreetMap, die mit einem Natrium-Leuchtmitteltyp erfasst sind. Die Anwendung verwendet Leaflet für die Karte und fragt die Overpass-API direkt aus dem Browser ab.

Der sichtbare Kartenausschnitt startet bei München. Die aktuelle Overpass-Abfrage ist jedoch nicht auf München oder ein anderes geografisches Gebiet eingeschränkt.

## 2. Projektinhalt

```text
sox-map/
├── sox-map.html   # Originaldatei, unverändert übernommen
└── readme.md      # Diese technische Beschreibung
```

Die HTML-Datei wurde bytegenau aus der bereitgestellten Datei übernommen.

## 3. Technischer Aufbau

### 3.1 HTML-Grundstruktur

Die Datei ist eine vollständige Single-Page-Anwendung ohne Build-Prozess:

- `<!doctype html>` und deutsche Seitensprache `lang="de"`
- responsives Viewport-Meta-Element
- Leaflet-CSS und Leaflet-JavaScript werden von `unpkg.com` geladen
- eine Vollbildkarte mit `id="map"`
- ein absolut positioniertes Bedienpanel mit:
  - Statusanzeige
  - Schaltfläche zum Laden der Daten
  - Schaltfläche zum Öffnen in Overpass Turbo
  - Statistikbereich
  - Hinweis zur Datenqualität von OpenStreetMap

Die Anwendung kann grundsätzlich direkt als HTML-Datei geöffnet werden. Ob die externen Requests aus `file://` funktionieren, hängt vom Browser und den CORS-/Referrer-Regeln der angesprochenen Dienste ab. Ein lokaler Webserver ist daher für zuverlässige Nutzung empfehlenswert.

### 3.2 Karteninitialisierung

Die Karte wird mit folgenden Startparametern erstellt:

```js
const map = L.map('map').setView([48.137, 11.575], 11);
```

Das entspricht dem Raum München. Als Hintergrundkarte werden OpenStreetMap-Kacheln über folgende URL verwendet:

```text
https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png
```

Die OpenStreetMap-Mitwirkenden werden über die Leaflet-Attribution genannt. Die Treffer werden in einer gemeinsamen `L.layerGroup()`-Gruppe gespeichert, damit sie vor einem erneuten Ladevorgang gelöscht werden können.

## 4. Overpass-Abfrage

Die Konstante `Q` enthält eine Overpass-QL-Abfrage:

```overpass
[out:json][timeout:900];
(
  nwr["highway"="street_lamp"]
     ["lamp_type"~"^low_pressure_sodium$|^low-pressure_sodium$|^SOX$|^sodium_vapor$",i];

  nwr["highway"="street_lamp"]
     ["light:method"~"^low_pressure_sodium$|^low-pressure_sodium$|^SOX$|^sodium_vapor$",i];

  nwr["highway"="street_lamp"]
     ["lamp_model"~"SOX|low.?pressure.?sodium",i];
);

out center;
```

### 4.1 Gesuchte Objekte

Die Abfrage sucht `node`, `way` und `relation` (`nwr`) mit:

- `highway=street_lamp`
- einem passenden `lamp_type`
- oder einem passenden `light:method`
- oder einer passenden `lamp_model`-Bezeichnung

Berücksichtigt werden unter anderem:

- `low_pressure_sodium`
- `low-pressure_sodium`
- `SOX`
- `sodium_vapor`
- Modellbezeichnungen mit `SOX`
- Modellbezeichnungen mit Varianten von `low pressure sodium`

Das `i` am Ende der regulären Ausdrücke aktiviert die Groß-/Kleinschreibungsunabhängigkeit in Overpass.

### 4.2 Wichtiger funktionaler Befund: keine München-Begrenzung

Die Kommentare im Code verweisen auf die Münchner Verwaltungsrelation `62428` beziehungsweise auf die Overpass-Area `3600062428`. Diese Area wird in der eigentlichen Abfrage aber nicht verwendet.

Dadurch fragt die Anwendung nicht nur München, sondern prinzipiell alle weltweit passenden Straßenlampen ab. Das hat mehrere Folgen:

- deutlich größere Antwortmengen
- längere Laufzeiten
- höhere Belastung der Overpass-Instanz
- mögliche Timeouts oder Rate-Limits
- falsche Aussage im Titel und im Bedienpanel
- `map.fitBounds(...)` kann die Karte nach dem Laden auf einen weltweiten Bereich zoomen

Für eine München-Karte müsste jede Abfrage beispielsweise mit einer Area-Einschränkung versehen werden:

```overpass
area(3600062428)->.munich;
nwr(area.munich)["highway"="street_lamp"] ...;
```

Dabei sollte die Area-ID vor dem produktiven Einsatz nochmals gegen die gewünschte Münchner Verwaltungsgrenze geprüft werden.

## 5. Netzwerk- und Fallback-Logik

Es werden zwei Overpass-Endpunkte nacheinander probiert:

1. `overpass.private.coffee`
2. `overpass-api.de`

Die Abfrage wird als URL-Parameter `data=...` per HTTP-GET übertragen. Für jeden Request gilt:

- `Accept: application/json`
- `cache: no-store`
- Abbruch nach 175 Sekunden über `AbortController`
- Prüfung des HTTP-Status
- JSON-Parsing

Wenn der erste Endpunkt fehlschlägt, wird der zweite probiert. Erst wenn beide fehlschlagen, werden die gesammelten Fehlermeldungen im Statusbereich angezeigt.

Die Overpass-Abfrage selbst hat eine serverseitige Timeout-Angabe von 900 Sekunden, während der Browser-Request nach 175 Sekunden abgebrochen wird. Diese Werte sind daher nicht aufeinander abgestimmt: Der Browser wartet wesentlich kürzer als der in Overpass angegebene Maximalwert.

## 6. Verarbeitung der Antwort

Nach dem erfolgreichen Request wird `data.elements` gelesen. Die Anwendung verarbeitet anschließend ausschließlich Elemente mit:

```js
e.type === "node" && e.lat != null && e.lon != null
```

Das bedeutet:

- echte OSM-Knoten werden dargestellt
- Ways und Relations werden trotz `out center` verworfen
- die von Overpass gelieferten Center-Koordinaten für Ways und Relations werden nicht genutzt

Der Overpass-Ausgabemodus `out center` ist deshalb aktuell nur teilweise sinnvoll. Entweder sollte die Anwendung Ways und Relations ebenfalls darstellen oder die Abfrage sollte nur Nodes anfordern.

## 7. Typklassifizierung

Die Funktion `typeOf(tags)` kombiniert diese Tag-Werte:

- `lamp_type`
- `light:method`
- `light_source`

Danach wird die Zeichenkette kleingeschrieben und klassifiziert:

- `low_pressure_sodium` oder `sox` → `Natrium-Niederdruck`
- `high_pressure_sodium`, `hps` oder `son` → `Natrium-Hochdruck`
- sonst → `Natrium`

### 7.1 Klassifizierungsproblem bei `lamp_model`

Die Overpass-Abfrage sucht auch über `lamp_model`. Die Funktion `typeOf()` wertet `lamp_model` jedoch nicht aus.

Ein Treffer, der ausschließlich wegen `lamp_model=SOX...` gefunden wurde, wird daher nicht über das Modell als Niederdruck erkannt, sondern kann als allgemeines `Natrium` klassifiziert werden.

Robuster wäre beispielsweise:

```js
const s = [
  t.lamp_type,
  t["light:method"],
  t.light_source,
  t.lamp_model
].filter(Boolean).join("|").toLowerCase();
```

### 7.2 Hochdruck-Zählung ist praktisch nicht passend zur Abfrage

Die Overpass-Abfrage sucht ausdrücklich Niederdruck-/SOX-Muster. Hochdruckbegriffe wie `high_pressure_sodium`, `HPS` oder `SON` sind dort nicht als eigene Suchbedingung enthalten.

Der Hochdruck-Zweig in `typeOf()` ist deshalb unter normalen Datenbedingungen kaum erreichbar. Für eine echte Hochdruck-Kategorie müssten passende Suchmuster in die Overpass-Abfrage aufgenommen werden.

## 8. Darstellung der Marker

Für jeden verarbeiteten Knoten wird ein `L.circleMarker` erzeugt und zur Layergruppe hinzugefügt.

### Hochdruck

```js
radius: 5,
weight: 1,
fillOpacity: .9
```

Da keine Farbe gesetzt wird, verwendet Leaflet seine Standardfarben.

### Niederdruck

```js
radius: 5,
weight: 1,
color: "#000000",
fillColor: "#FFD700",
fillOpacity: 0.9
```

Niederdruck wird gelb mit schwarzem Rand dargestellt.

### Allgemeines Natrium

Auch hier werden nur Standardwerte verwendet. Dadurch sind die drei Kategorien visuell nicht eindeutig unterscheidbar.

Zusätzlich sind die `<i>`-Elemente in der Legende zwar als Kreise formatiert, erhalten aber keine individuellen Farben. Die Legende erklärt daher die Kategorien, zeigt aber keine korrespondierenden Farbsymbole.

## 9. Popups und HTML-Escaping

Die Popup-Funktion listet alle OSM-Tags tabellarisch auf. Werte werden mit `esc()` gegen HTML-Sonderzeichen geschützt. Das ist wichtig, weil OSM-Tags externe oder unerwartete Zeichen enthalten können.

Der erzeugte Link lautet immer:

```text
https://www.openstreetmap.org/node/{id}
```

Da aktuell nur Nodes verarbeitet werden, ist dieser Link für die tatsächlich angezeigten Elemente korrekt. Falls später Ways oder Relations dargestellt werden, muss der OSM-Pfad dynamisch aus `e.type` gebildet werden.

Das `target="_blank"`-Attribut sollte aus Sicherheitsgründen um `rel="noopener noreferrer"` ergänzt werden.

## 10. Statistik und Kartenzoom

Gezählt werden:

- `hp`: Natrium-Hochdruck
- `lp`: Natrium-Niederdruck
- `other`: Natrium ohne genauere Typzuordnung

Die Statusanzeige meldet die Anzahl der erzeugten Marker. Diese Anzahl entspricht nicht zwingend der Zahl aller Overpass-Elemente, weil Ways und Relations vorher verworfen werden.

Bei mindestens einem Marker wird die Karte auf die Treffergrenzen gesetzt:

```js
map.fitBounds(L.featureGroup(markers).getBounds().pad(.04));
```

Bei einer nicht eingeschränkten globalen Query kann dieser Zoom die eigentliche München-Ansicht vollständig verlassen.

## 11. Overpass-Turbo-Schaltfläche

Die Schaltfläche öffnet Overpass Turbo und übergibt die Query über den URL-Parameter `Q`.

Als Styling wird jedoch diese Regel übergeben:

```text
{{style:
node[natural=sodium];
}}
```

Die aktuelle Overpass-Abfrage sucht nicht nach `natural=sodium`. Das Styling passt daher nicht zur tatsächlichen Trefferlogik und wird voraussichtlich keine sinnvolle farbliche Darstellung der geladenen Straßenlampen erzeugen.

Das Styling sollte sich auf die tatsächlich verwendeten Tags oder auf eine passende Darstellung aller Query-Ergebnisse beziehen.

## 12. Sicherheits- und Robustheitsaspekte

Positiv:

- OSM-Tag-Inhalte werden vor der Einfügung ins Popup escaped.
- Es werden keine Benutzerpasswörter oder lokalen Geheimnisse verwendet.
- Der Request besitzt einen clientseitigen Abbruchmechanismus.
- Bei Fehlern werden mehrere Endpunkte versucht.

Verbesserungspunkte:

- globale Query durch eine Area- oder Bounding-Box-Einschränkung ersetzen
- `rel="noopener noreferrer"` bei externen Links ergänzen
- gleichzeitige Mehrfachausführung von `load()` verhindern, z. B. durch Deaktivieren der Schaltfläche während des Requests
- Retry-/Backoff-Strategie bei temporären Overpass-Fehlern ergänzen
- Antwortgröße begrenzen oder Ergebnisse paginieren, falls große Datenmengen erwartet werden
- Endpunktverfügbarkeit und CORS-Verhalten für den tatsächlichen Browserbetrieb testen
- Ergebnisse gegebenenfalls deduplizieren, falls sich Query-Selektoren bei Änderungen überschneiden

## 13. Empfohlene nächste Änderungen

Priorität 1:

1. München tatsächlich in der Overpass-Abfrage filtern.
2. `lamp_model` in die Klassifizierung aufnehmen.
3. Query-Muster und Kategorien konsistent machen: entweder nur Niederdruck/SOX anzeigen oder Hochdruck explizit abfragen.
4. Die Overpass-Turbo-Stildefinition an die verwendeten Tags anpassen.

Priorität 2:

1. Farben für Marker und Legende vereinheitlichen.
2. OSM-Links dynamisch nach Elementtyp erzeugen.
3. Ways und Relations entweder verarbeiten oder in der Query ausschließen.
4. Mehrfachklicks und parallele Requests verhindern.

Priorität 3:

1. Ladefortschritt und Antwortgröße anzeigen.
2. Einen lokalen Testmodus mit gespeicherten Beispieldaten ergänzen.
3. Eine Such- oder Gebietsauswahl für weitere Städte einbauen.
4. Für große Abfragen einen eigenen Backend-Proxy oder eine vorab erzeugte GeoJSON-Datei verwenden.

## 14. Nutzung

Die Datei kann direkt geöffnet werden:

```text
/root/sox-map/sox-map.html
```

Für einen lokalen HTTP-Test kann aus dem Projektordner vorübergehend ein einfacher Webserver gestartet werden:

```bash
cd /root/sox-map
python3 -m http.server 8000
```

Danach ist die Anwendung unter `http://127.0.0.1:8000/sox-map.html` erreichbar. Der Server sollte nach dem Test wieder beendet werden, sofern er nicht ausdrücklich dauerhaft benötigt wird.

## 15. Verifizierter Stand

- Projektordner angelegt: `/root/sox-map`
- Originaldatei kopiert: `/root/sox-map/sox-map.html`
- Analyse erstellt: `/root/sox-map/ANALYSE.md`
- SHA-256 der Quell- und Zieldatei identisch:

```text
4f208f5e5b7a91639cd925d603a4d60f93ebc5c303c464503d7e469a44f4cd9a
```

Es wurde keine inhaltliche Änderung an der HTML-Datei vorgenommen.
