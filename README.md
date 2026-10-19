# SOX Map

Interaktive Leaflet-Karte für weltweit in OpenStreetMap erfasste Straßenlampen mit Natrium-Leuchtmitteln.

Die Anwendung läuft als einzelne HTML-Datei. Sie fragt die Overpass API direkt im Browser ab und stellt passende OSM-Knoten auf einer OpenStreetMap-Karte dar.

## Funktionen

- globale Suche ohne regionale Begrenzung
- Darstellung von `highway=street_lamp`
- Erkennung über `lamp_type`, `light:method`, `light_source` und `lamp_model`
- Kategorien mit einheitlichen Farben:
  - Orange: Natrium-Hochdruck
  - Rot: Natrium-Niederdruck
  - Schwarz: Typ unbekannt
- Popups mit den vollständigen OSM-Tags
- direkter Link zum jeweiligen OSM-Objekt
- Anzeige des Ladezustands während der Overpass-Abfrage
- Anzeige von:
  - Antwortgröße
  - Anzahl der gelieferten OSM-Elemente
  - Anzahl der dargestellten Knoten
  - Trefferzahl je Leuchtmittelkategorie
- automatischer Fallback auf einen zweiten Overpass-Endpunkt
- Öffnen der verwendeten Abfrage in Overpass Turbo

## Dateien

```text
sox-map/
├── sox-map.html
└── README.md
```

## Verwendung

### Direkt im Browser

`sox-map.html` kann direkt geöffnet werden. Je nach Browser können Requests von `file://` zu externen APIs durch CORS- oder Referrer-Regeln eingeschränkt sein.

### Mit einem lokalen Webserver

Für einen zuverlässigen Test empfiehlt sich ein lokaler HTTP-Server:

```bash
cd /root/sox-map
python3 -m http.server 8000
```

Danach im Browser öffnen:

```text
http://127.0.0.1:8000/sox-map.html
```

Der Server kann mit `Ctrl+C` beendet werden.

## Datenabfrage

Die Anwendung verwendet eine globale Overpass-QL-Abfrage. Es gibt keine regionale, Länder- oder Bounding-Box-Begrenzung.

Gesucht werden Straßenlampen mit folgenden Merkmalen:

- `lamp_type`
- `light:method`
- `light_source`
- `lamp_model`

Berücksichtigte Bezeichnungen umfassen unter anderem:

- `low_pressure_sodium`
- `low-pressure_sodium`
- `SOX`
- `sodium_vapor`
- `high_pressure_sodium`
- `high-pressure_sodium`
- `HPS`
- `SON`

Die Abfrage verwendet:

```text
[out:json][timeout:120]
```

Der Browser bricht einen Request nach 130 Sekunden ab. Die zehn Sekunden Differenz geben dem Browser Zeit für die Antwortverarbeitung, ohne den Overpass-Timeout unnötig weit auseinanderzuziehen.

## Overpass-Endpunkte

Die Endpunkte werden in dieser Reihenfolge probiert:

1. `https://overpass.private.coffee/api/interpreter`
2. `https://overpass-api.de/api/interpreter`

Wenn der erste Endpunkt fehlschlägt, wird automatisch der zweite verwendet. Schlagen beide Requests fehl, zeigt die Oberfläche die jeweiligen Fehlermeldungen an.

## Darstellung

Marker werden als Leaflet-Kreisobjekte gezeichnet:

- Hochdruck: orange Füllung mit dunkelorangefarbenem Rand
- Niederdruck: rote Füllung mit dunkelrotem Rand
- unbekannter Typ: schwarze Füllung und schwarzer Rand

Die Karte startet mit einer weltweiten Ansicht. Nach erfolgreichem Laden wird auf die Grenzen der gefundenen Knoten gezoomt.

Ways und Relations werden derzeit nicht als Marker dargestellt. Obwohl die Overpass-Abfrage `nwr` verwendet, verarbeitet die Oberfläche nur Elemente mit eigenen `lat`- und `lon`-Werten, also OSM-Nodes.

## Technische Hinweise

- Leaflet `1.9.4` wird von `unpkg.com` geladen.
- Die Hintergrundkacheln stammen von OpenStreetMap.
- OSM-Tag-Werte werden vor der Verwendung in Popups HTML-escaped.
- Die Antwortgröße wird nach dem Empfang des vollständigen JSON-Texts in Bytes ermittelt.
- Der sichtbare Ladebalken zeigt den aktiven Request an. Die Overpass API liefert keinen verlässlichen prozentualen Fortschritt, daher wird kein erfundener Prozentwert angezeigt.
- Große globale Abfragen können lange dauern oder von Overpass-Instanzen abgelehnt werden. Die Anwendung ist von der Verfügbarkeit und den Nutzungsbedingungen der öffentlichen Endpunkte abhängig.

## Datenqualität

OpenStreetMap ist gemeinschaftlich gepflegt. Fehlende Treffer bedeuten nicht, dass eine reale Straßenlampe kein Natrium-Leuchtmittel verwendet. Angezeigt werden ausschließlich Objekte, deren OSM-Tags mit der Abfrage übereinstimmen.

## Lizenz und Quellen

- Kartendaten: [OpenStreetMap](https://www.openstreetmap.org/copyright)
- Kartenbibliothek: [Leaflet](https://leafletjs.com/)
- Abfrageschnittstelle: [Overpass API](https://wiki.openstreetmap.org/wiki/Overpass_API)

Die jeweiligen Nutzungsbedingungen und Attributionsanforderungen der externen Dienste sind zu beachten.
