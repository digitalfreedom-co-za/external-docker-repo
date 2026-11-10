# epilot Dynamic Backend betreiben

## 1. Lizenz und Voraussetzungen

Die Nutzung setzt eine gültige Lizenz von DigitalFreedom Global LLC voraus. Sie benötigen Docker, Zugang zum freigegebenen Containerimage, eine erreichbare PostgreSQL Datenbank ab Version 15 oder Oracle Database ab 23ai 23.6 sowie einen HTTPS Reverse Proxy für den Zugriff aus epilot. Das Image enthält keine kundenspezifischen Endpunkte. Ein Neubau ist für deren Einrichtung nicht erforderlich.

## 2. Image laden

Diese Anleitung verwendet die festgelegte Version `0.0.1`. Die Version muss zuvor veröffentlicht worden sein. Wählen Sie genau die Variante Ihrer Datenbank:

```bash
docker pull ghcr.io/digitalfreedom-co-za/external-docker-repo/epilot-dynamic-backend:0.0.1-postgresql
```

Für Oracle ersetzen Sie `0.0.1-postgresql` durch `0.0.1-oracle`. Falls der Registryzugang eine Anmeldung verlangt, verwenden Sie `docker login ghcr.io` mit dem bereitgestellten Zugang. Geben Sie Zugangsdaten nicht als Kommandozeilenargumente weiter.

## 3. Datenbank verbinden

Legen Sie eine eigene Datenbank beziehungsweise ein eigenes Oracle Schema an. Der Benutzer benötigt Rechte zum Erstellen und Ändern der Tabellen, Indizes und Constraints sowie zum Lesen und Schreiben der Daten. Unter Oracle benötigt er außerdem ein Kontingent auf seinem Tablespace. Liquibase legt beim ersten Start das Schema an und führt spätere Migrationen aus.

Erstellen Sie eine Datei `backend.env`, die nur der Betreiber lesen darf. Hinterlegen Sie darin die tatsächlichen Zugangsdaten über Ihren sicheren Bereitstellungsprozess:

```dotenv
DB_JDBC_URL=jdbc:postgresql://datenbank:5432/epilot
DB_USERNAME=epilot
DB_PASSWORD=<Datenbankpasswort>
```

Für Oracle lautet die URL beispielsweise `jdbc:oracle:thin:@//datenbank:1521/FREEPDB1`. Der Datenbankhost muss aus dem Container erreichbar sein. `localhost` bezeichnet dort den Container selbst. Die Datenbankvariante des Images ist fest vorgegeben und lässt sich nicht durch eine Umgebungsvariable ändern.

## 4. Endpunkte konfigurieren

Erstellen Sie neben `backend.env` eine Datei `endpoints.yaml`:

```yaml
app:
  endpoints:
    - path: tarife
      type: dropdown
    - path: netz-produkt-tarif
      type: cascade
      levels: 3
```

Mindestens ein Endpunkt muss konfiguriert sein. Pfade müssen eindeutig sein. Erlaubt sind Buchstaben und Ziffern sowie `-` und `_` innerhalb eines Pfadsegments. `/` trennt Segmente, darf aber weder am Anfang noch am Ende stehen. Ein Dropdown hat keine Angabe `levels`. Eine Kaskade benötigt `levels: 2` oder `levels: 3`. Ungültige Angaben verhindern den Start.

Starten Sie den Container aus diesem Verzeichnis:

```bash
docker run -d --name epilot-dynamic-backend \
  --restart unless-stopped \
  --env-file backend.env \
  -e QUARKUS_CONFIG_LOCATIONS=file:/config/endpoints.yaml \
  --mount type=bind,src="$PWD/endpoints.yaml",dst=/config/endpoints.yaml,readonly \
  -p 127.0.0.1:8080:8080 \
  ghcr.io/digitalfreedom-co-za/external-docker-repo/epilot-dynamic-backend:0.0.1-postgresql
```

Für Oracle verwenden Sie wieder das Image mit `0.0.1-oracle`. Richten Sie den Reverse Proxy mit HTTPS auf Port 8080 ein. Bei einem Reverse Proxy in einem separaten Container verwenden Sie ein gemeinsames Docker Netzwerk und den Backend Containernamen als Ziel.

Nach Änderungen an `endpoints.yaml` starten Sie den Container neu. Bei atomarem Austausch der Datei erstellen Sie den Container neu, damit der Dateimount die neue Datei verwendet. Entfernte Endpunkte werden inaktiv und liefern HTTP 404; gespeicherte Einträge bleiben erhalten. Alle Replikate benötigen dieselbe Konfiguration. Unter Oracle starten Sie bei neuen Endpunkten zuerst ein Replikat und skalieren erst nach erfolgreicher Synchronisation.

Alternativ können Sie die Liste über Standardvariablen wie `APP_ENDPOINTS_0__PATH=tarife` und `APP_ENDPOINTS_0__TYPE=dropdown` konfigurieren. Beginnen Sie mit Index 0 und verwenden Sie fortlaufende Indizes. Verwenden Sie eine vollständige Quelle für die Liste, da Quarkus Einträge aus mehreren Quellen nach Index zusammenführt.

## 5. Daten laden

Starten Sie den Dienst einmal mit der Endpunktkonfiguration. Dadurch entstehen die Einträge in `option_endpoint`. Laden Sie danach Ihre Daten mit Ihrem SQL Werkzeug in `option_item`. Es gibt keinen HTTP Importendpunkt. Die folgenden Beispiele funktionieren für beide Datenbanken. Führen Sie den Import in einer Transaktion aus und bestätigen Sie diese nach Prüfung mit `COMMIT`.

```sql
INSERT INTO option_item (endpoint_path, item_level, item_key, label, item_value, info, sort_order)
VALUES ('tarife', 1, 'basis', 'Basis', 'BASIS', '12 Monate Laufzeit', 1);

INSERT INTO option_item (endpoint_path, item_level, item_key, label, item_value, sort_order)
VALUES ('netz-produkt-tarif', 1, 'nord', 'Netz Nord', 'NORD', 1);

INSERT INTO option_item (endpoint_path, parent_id, item_level, item_key, label, item_value, sort_order)
SELECT endpoint_path, id, 2, 'strom', 'Strom', 'STROM', 1
FROM option_item
WHERE endpoint_path = 'netz-produkt-tarif' AND item_level = 1 AND item_value = 'NORD';

INSERT INTO option_item (endpoint_path, parent_id, item_level, item_key, label, item_value, sort_order)
SELECT endpoint_path, id, 3, 'basis', 'Basis', 'BASIS', 1
FROM option_item
WHERE endpoint_path = 'netz-produkt-tarif' AND item_level = 2 AND item_value = 'STROM';

COMMIT;
```

Führen Sie diese Beispiele nur einmal in einer neuen Liste aus. `item_key` und `item_value` müssen je Endpunkt und Elternknoten eindeutig sein. Jeder Kindknoten liegt genau eine Ebene unter seinem Elternknoten. Das Löschen eines Knotens löscht auch dessen Kinder. Deaktivieren mit `active = FALSE` erhält die Daten. Verwenden Sie für Pflichtfelder keine leeren Zeichenketten; Oracle behandelt diese als NULL.

Aktive Einträge werden nach `sort_order`, dann `label` und `id` sortiert. Pro Antwort werden höchstens 1000 Einträge geliefert. `info` ist optional und fehlt bei NULL in der JSON Antwort. Datenänderungen benötigen keinen Neustart.

## 6. Optionalen Zugriffsschutz aktivieren

Ergänzen Sie in `backend.env` die Variable `APP_AUTH_BEARER_TOKEN` mit einem geheimen Token und erstellen Sie den Container mit derselben Startanweisung neu. Ohne Token oder bei leerem Wert ist die API öffentlich. Bei gesetztem Token verlangen API Aufrufe den Header `Authorization: Bearer <Token>`. Der Dienst protokolliert den Token nicht. Die Health Endpunkte bleiben ohne Authentifizierung erreichbar.

## 7. Betrieb prüfen

```bash
curl --fail http://localhost:8080/q/health/ready
curl --fail http://localhost:8080/q/health/live
```

`/q/health` fasst die Prüfungen zusammen; `/q/health/started` prüft den Startzustand. Warten Sie beim ersten Start auf erfolgreiche Migration und Datenbankverbindung. Prüfen Sie anschließend `/api/v1/tarife` und die Kaskade mit Ihrem HTTP Werkzeug, bei aktiviertem Schutz mit dem Authorization Header. Ohne Einträge ist `[]` eine gültige Antwort.

Eine erfolgreiche Antwort enthält Objekte wie:

```json
[{"key":"basis","label":"Basis","value":"BASIS","info":"12 Monate Laufzeit"}]
```

HTTP 401 bedeutet fehlende oder falsche Authentifizierung. HTTP 404 bedeutet einen unbekannten oder inaktiven Endpunkt beziehungsweise einen nicht gefundenen Elternwert. HTTP 400 weist auf ungültige, doppelte oder unerwartete Parameter hin. Dropdowns erlauben keine Queryparameter. Bei Kaskaden muss `level1` vor `level2` angegeben werden; eine Kaskade mit zwei Ebenen erlaubt kein `level2`.

## 8. epilot Apps einrichten

Öffnen Sie in epilot `Konfiguration`, `Apps`, `Installierte Apps` und die App-Einstellungen der jeweiligen App. Beide Apps verwenden den epilot API Proxy:

| Einstellung | Wert |
|---|---|
| `api_host` | Öffentlicher Hostname des Backends ohne `https://` |
| `secret` | Derselbe Wert wie `APP_AUTH_BEARER_TOKEN`, bei öffentlichem Zugriff leer |

Der Proxy setzt `https://`, Hostname und konfigurierten Pfad zusammen. Das Zertifikat muss gültig und der Host erreichbar sein.

Für den Dynamic Dropdown Block:

| Einstellung | Wert |
|---|---|
| `api_path` | `/api/v1/tarife` |

Für den Dynamic Cascade Block:

| Einstellung | Wert |
|---|---|
| `level1_source` | `/api/v1/netz-produkt-tarif` |
| `level2_source` | `/api/v1/netz-produkt-tarif?level1={1}` |
| `level3_source` | `/api/v1/netz-produkt-tarif?level1={1}&level2={2}` |

Bei zwei Ebenen entfällt `level3_source`. Die Standardfelder passen zur Antwort, daher bleiben `items_path` und die `*_field` Argumente leer. Testen Sie im Journey die Auswahl jeder Ebene. Werte in Queryparametern müssen URL kodiert sein.

## 9. Version gezielt aktualisieren

Sichern Sie die Datenbank und Ihre Konfiguration. Lesen Sie die Versionshinweise und testen Sie die neue Version mit einer separaten Datenbank in Ihrer Testumgebung. Laden Sie das Image mit der exakten neuen Versionsnummer und demselben Datenbanksuffix. Verwenden Sie für reproduzierbare Updates keinen `latest` Tag.

Stoppen und entfernen Sie anschließend den bisherigen Container und führen Sie den Startbefehl aus Schritt 4 mit der neuen Versionsnummer aus. Bewahren Sie die externe Datenbank und Konfigurationsdateien auf. Prüfen Sie Health Endpunkte und repräsentative Dropdowns und Kaskaden. Ein Zurücksetzen des Images setzt bei inkompatiblen Datenbankmigrationen auch das Wiederherstellen einer passenden Sicherung voraus.

Diese Anleitung wird bei jeder externen Veröffentlichung automatisch im Kundenrepository unter `epilot-dynamic-backend/README.md` aktualisiert.
