# epilot Dynamic Backend betreiben

Die Apps können jede API abfragen, die Daten in der Struktur der Dynamic Apps liefert. Alternativ nutzen Sie das kostenlos enthaltene Backend `epilot-dynamic-backend` und müssen dann für die Bereitstellung der Daten nur die Tabellen befüllen. Beide Möglichkeiten sind gleichwertig. Diese Anleitung beschreibt das Selbsthosting des optionalen, kostenlos enthaltenen Backends. Der View-Modus ist eine fortgeschrittene Option.

## Verfügbare Tags

Diese Liste wird bei jeder Veröffentlichung automatisch erzeugt.

| Version | Exakte Tags |
| --- | --- |
| 0.1.0 | `0.1.0-oracle`, `0.1.0-oracle-native`, `0.1.0-postgresql`, `0.1.0-postgresql-native` |
| 0.0.1 | `0.0.1-oracle`, `0.0.1-postgresql` |

Mitlaufende Tags zeigen auf den zuletzt veröffentlichten exakten Tag mit derselben Versionslinie und demselben Suffix.
Verwenden Sie im Produktivbetrieb immer einen exakten Tag.

| Mitlaufender Tag | Zeigt auf |
| --- | --- |
| `latest-oracle` | `0.1.0-oracle` |
| `latest-oracle-native` | `0.1.0-oracle-native` |
| `latest-postgresql` | `0.1.0-postgresql` |
| `latest-postgresql-native` | `0.1.0-postgresql-native` |
| `0.1-oracle` | `0.1.0-oracle` |
| `0.1-oracle-native` | `0.1.0-oracle-native` |
| `0.1-postgresql` | `0.1.0-postgresql` |
| `0.1-postgresql-native` | `0.1.0-postgresql-native` |
| `0.0-oracle` | `0.0.1-oracle` |
| `0.0-postgresql` | `0.0.1-postgresql` |

## 1. Lizenz und Voraussetzungen

Die Nutzung setzt eine gültige Lizenz von DigitalFreedom Global LLC voraus. Sie benötigen Docker, Zugang zum freigegebenen Containerimage, eine erreichbare PostgreSQL Datenbank ab Version 15 oder Oracle Database ab 23ai 23.6 sowie einen HTTPS-Reverse-Proxy für den Zugriff aus epilot. Das Image enthält keine kundenspezifischen Endpunkte. Ein Neubau ist für deren Einrichtung nicht erforderlich.

## 2. Image laden

Die Beispiele für `docker pull` und `docker run` verwenden die feste Beispielversion `0.0.1`; ersetzen Sie diese durch die aktuelle Releaseversion. Wählen Sie genau die Variante Ihrer Datenbank:

```bash
docker pull ghcr.io/digitalfreedom-co-za/external-docker-repo/epilot-dynamic-backend:0.0.1-postgresql
```

Für Oracle ersetzen Sie `0.0.1-postgresql` durch `0.0.1-oracle`. Falls der Registryzugang eine Anmeldung verlangt, verwenden Sie `docker login ghcr.io` mit dem bereitgestellten Zugang. Geben Sie Zugangsdaten nicht als Kommandozeilenargumente weiter.

### JVM oder Native wählen

Jede externe Veröffentlichung liefert vier Varianten. Native Images enthalten ein vorab kompiliertes Programm und benötigen keine JVM im Container. Beide Buildarten unterstützen Tabellenmodus, View-Modus, externe `endpoints.yaml` und die Health Endpunkte unverändert.

| Datenbank | JVM Tags | Native Tags |
|---|---|---|
| PostgreSQL | `X.Y.Z-postgresql`, `X.Y-postgresql`, `latest-postgresql` | `X.Y.Z-postgresql-native`, `X.Y-postgresql-native`, `latest-postgresql-native` |
| Oracle | `X.Y.Z-oracle`, `X.Y-oracle`, `latest-oracle` | `X.Y.Z-oracle-native`, `X.Y-oracle-native`, `latest-oracle-native` |

JVM bleibt die Vorgabe. Wählen Sie JVM für lang laufende Dienste mit hohem Durchsatz und zur Nutzung etablierter JVM Diagnosewerkzeuge. Die Laufzeitoptimierung der JVM kann nach einer Aufwärmphase einen höheren Spitzendurchsatz erreichen. Native eignet sich besonders für häufige Neustarts, kurze Laufzeiten und knappen Arbeitsspeicher. Native startet typischerweise schneller, benötigt häufig weniger Speicher und liefert kleinere Images, braucht beim Bauen jedoch wesentlich mehr Zeit und Arbeitsspeicher. Spitzendurchsatz und Speicherbedarf hängen von Datenbank, Abfragen und Last ab; sie wurden hier nicht als Lastbenchmark gemessen.

Für einen Native Start hängen Sie `-native` an den vollständigen Datenbanktag an, beispielsweise `0.0.1-postgresql-native`. Der Abschnitt `Verfügbare Tags` wird bei der nächsten externen Veröffentlichung automatisch mit den tatsächlich veröffentlichten Tags gefüllt. Die Beispiele kündigen keine bereits verfügbare Version an.

### Laufzeitvoraussetzungen je Variante

Das JVM Image enthält die Java Laufzeit aus `eclipse-temurin:25-jre`. Das Native Image basiert auf `registry.access.redhat.com/ubi9/ubi-minimal:9.8` (ubi9 / ubi-minimal) und benötigt keine JVM im Container. Die Releaseimages sind für Linux AMD64 gebaut. Native läuft als UID 1001, JVM derzeit als root; beachten Sie die Dateirechte aus Abschnitt 4.

Planen Sie bei JVM Speicher für Heap und weitere Prozessallokationen ein, bei Native für den Prozess und seine nativen Allokationen. Lassen Sie bei beiden Varianten Reserve oberhalb der unten beobachteten RSS Werte. Messen Sie Spitzenbedarf und CPU Auslastung mit Ihren Abfragen und Ihrer erwarteten Last, bevor Sie Containerlimits festlegen. Planen Sie die Datenbank separat. Die Messumgebung hatte 4 virtuelle CPUs und 8 GB Docker RAM; dies ist keine Mindestanforderung je Container.

### Messwerte dieses Builds

Gemessen auf diesem Build am 11.11.2026: Linux AMD64 Images unter Docker Desktop auf einem ARM64 Mac mit AMD64 Emulation, 4 virtuellen CPUs und 8 GB Docker RAM, Quarkus 3.34.7, PostgreSQL 17 und Oracle Free 23.6. Die lokalen Prüfimages wurden mit Mandrel 25.0.5 gebaut. Der Release Workflow verwendet dagegen in CI das GraalVM Setup aus org-ci-config mit Oracle GraalVM 25. Alle vier Images bestanden beide Betriebsmodi, dynamische Endpunkte aus einer Datei, Dropdowns, dreistufige Kaskaden, JSON Fehlerantworten und alle drei Health Endpunkte mit isolierten E2E-TEST Daten.

| Datenbank | Buildart | Imagegröße | Start Tabellenmodus | Start View-Modus | RSS Tabellenmodus | RSS View-Modus |
|---|---|---|---|---|---|---|
| PostgreSQL | JVM | 572.0 MiB | 8.551 s | 4.605 s | 427.4 MiB | 370.7 MiB |
| PostgreSQL | Native | 309.7 MiB | 2.238 s | 0.924 s | 241.2 MiB | 135.4 MiB |
| Oracle | JVM | 572.0 MiB | 9.413 s | 5.068 s | 449.1 MiB | 384.4 MiB |
| Oracle | Native | 310.4 MiB | 4.123 s | 1.046 s | 251.6 MiB | 148.2 MiB |

Start und Speicher wurden unter AMD64 Emulation auf einem ARM64 Mac gemessen. Die Emulation beeinflusst JVM und Native unterschiedlich; das Verhältnis kann auf einem echten AMD64 Host abweichen.

Der RSS wurde auf diesem Build bei der erneuten Prüfung für jede Variante und jeden Modus aus `/proc/1/status` im Container erfasst, etwa zehn Sekunden nach erfolgreichem `/q/health/ready` und dem Einfügen der E2E-TEST Tabellendaten, vor API Testlast. Die Views waren vor dem Start mit E2E-TEST Daten vorbereitet. Die Werte umfassen den Anwendungsprozess, nicht die Datenbank; sie sind keine Spitzenwerte unter Last.

Die Größen sind entpackte Imagegrößen laut `docker image inspect`, keine Downloadgrößen. Die Zeiten sind einzelne Quarkus Logwerte `started in`, ohne parallelen Compilerlauf. Der Tabellenstart enthält die Liquibase Erstmigration in einem frischen Anwendungsschema. Der View Start verwendet eine vorbereitete View und ein reines Lesekonto. Datenbankstart und Image Download sind nicht enthalten. Die Werte sind keine Leistungszusage für andere Hardware oder Kundendatenbanken.

Die nativen Maven Builds benötigten hier 8 Minuten 31 Sekunden für PostgreSQL und 7 Minuten 33 Sekunden für Oracle, jeweils ohne erneute Tests nach bereits erfolgreicher Prüfung und einschließlich lokaler Vorbereitung sowie gegebenenfalls Image Downloads. Kunden laden fertige Images und benötigen dafür kein GraalVM. Der Spitzendurchsatz wurde nicht gemessen.

## 3. Datenbank verbinden

Im standardmäßigen Tabellenmodus (`APP_DATA_SOURCE=tables`) legen Sie eine eigene Datenbank beziehungsweise ein eigenes Oracle Schema an. Der Benutzer benötigt Rechte zum Erstellen und Ändern der Tabellen, Indizes und Constraints sowie zum Lesen und Schreiben der Daten. Unter Oracle benötigt er außerdem ein Kontingent auf seinem Tablespace. Liquibase legt beim ersten Start das Schema an und führt spätere Migrationen aus. Für den ausschließlich lesenden View-Modus gelten stattdessen die Rechte aus Abschnitt 6.

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

Native läuft als UID 1001, JVM derzeit als root. Die eingebundene Endpunktdatei muss für UID 1001 lesbar sein. Eine Datei mit Modus 600 und einem anderen Eigentümer kann mit JVM funktionieren, aber den Native Start verhindern. Enthält `endpoints.yaml` nur Endpunktdefinitionen, setzen Sie `chmod 644 endpoints.yaml`. Passwörter und Tokens gehören in die geschützte Umgebungsdatei; für `.env` gilt weiterhin `chmod 600 .env`. Enthält die Endpunktdatei Geheimnisse, erteilen Sie UID 1001 stattdessen über Eigentümerrechte oder eine ACL gezielt Lesezugriff.

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

Für Oracle verwenden Sie wieder das Image mit `0.0.1-oracle`. Richten Sie den HTTPS-Reverse-Proxy auf Port 8080 ein. Bei einem HTTPS-Reverse-Proxy in einem separaten Container verwenden Sie ein gemeinsames Docker Netzwerk und den Backend Containernamen als Ziel.

Nach Änderungen an `endpoints.yaml` starten Sie den Container neu. Bei atomarem Austausch der Datei erstellen Sie den Container neu, damit der Dateimount die neue Datei verwendet. Im Tabellenmodus werden entfernte Endpunkte inaktiv und liefern HTTP 404; gespeicherte Einträge bleiben erhalten. Alle Replikate benötigen dieselbe Konfiguration. Im Tabellenmodus unter Oracle starten Sie bei neuen Endpunkten zuerst ein Replikat und skalieren erst nach erfolgreicher Synchronisation.

Alternativ können Sie die Liste über Standardvariablen wie `APP_ENDPOINTS_0__PATH=tarife` und `APP_ENDPOINTS_0__TYPE=dropdown` konfigurieren. Beginnen Sie mit Index 0 und verwenden Sie fortlaufende Indizes. Verwenden Sie eine vollständige Quelle für die Liste, da Quarkus Einträge aus mehreren Quellen nach Index zusammenführt.

## 5. Mit Docker Compose und eigener Datenbank starten

Als Alternative zum Startbefehl in Abschnitt 4 können Sie Docker Compose verwenden. Speichern Sie das folgende vollständige Beispiel als `compose.yaml` in einem eigenen Betriebsverzeichnis. Es startet nur den Dienst mit Ihrer vorhandenen Datenbank aus Abschnitt 3. Das öffentliche Kundenpaket besteht aus dem Containerimage und dieser Anleitung. Eine separate Compose Datei oder Endpunktdatei wird dort nicht veröffentlicht.

```yaml
x-backend-environment: &backend-environment
  DB_JDBC_URL: ${DB_JDBC_URL:?DB_JDBC_URL setzen}
  DB_USERNAME: ${DB_USERNAME:?DB_USERNAME setzen}
  DB_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD setzen}
  APP_DATA_SOURCE: ${APP_DATA_SOURCE:-tables}
  APP_DEMO_DATA: ${APP_DEMO_DATA:-false}
  APP_VIEW_NAME: ${APP_VIEW_NAME:-option_data}
  APP_AUTH_BEARER_TOKEN: ${APP_AUTH_BEARER_TOKEN:-}
  QUARKUS_CONFIG_LOCATIONS: file:/config/endpoints.yaml

services:
  backend-postgresql:
    profiles: [postgresql]
    image: ghcr.io/digitalfreedom-co-za/external-docker-repo/epilot-dynamic-backend:${IMAGE_VERSION:?IMAGE_VERSION setzen}-postgresql${IMAGE_VARIANT:-}
    environment: *backend-environment
    volumes:
      - type: bind
        source: ${ENDPOINT_CONFIG_FILE:?ENDPOINT_CONFIG_FILE setzen}
        target: /config/endpoints.yaml
        read_only: true
        bind:
          create_host_path: false
    ports:
      - "127.0.0.1:8080:8080"

  backend-oracle:
    profiles: [oracle]
    image: ghcr.io/digitalfreedom-co-za/external-docker-repo/epilot-dynamic-backend:${IMAGE_VERSION:?IMAGE_VERSION setzen}-oracle${IMAGE_VARIANT:-}
    environment: *backend-environment
    volumes:
      - type: bind
        source: ${ENDPOINT_CONFIG_FILE:?ENDPOINT_CONFIG_FILE setzen}
        target: /config/endpoints.yaml
        read_only: true
        bind:
          create_host_path: false
    ports:
      - "127.0.0.1:8080:8080"
```

Setzen Sie `IMAGE_VERSION` in Ihrer lokalen `.env` auf die exakte verfügbare Version ohne Datenbank- oder Native Suffix, zum Beispiel `0.0.1`, bis `0.1.0` verfügbar ist. Ohne diese Angabe bricht Compose ab. Prüfen Sie die verfügbaren Tags aus Abschnitt 2.

Legen Sie dort auch `endpoints.yaml` im Format aus Abschnitt 4 an. Beachten Sie insbesondere bei `IMAGE_VARIANT=-native` die dort beschriebenen Leserechte für UID 1001. Erstellen Sie daneben eine Datei `.env` mit den folgenden Platzhaltern und ersetzen Sie die Werte durch Ihre Konfiguration:

```dotenv
DB_JDBC_URL=jdbc:postgresql://datenbank:5432/epilot
DB_USERNAME=epilot
DB_PASSWORD='<Datenbankpasswort>'
ENDPOINT_CONFIG_FILE=./endpoints.yaml
# Optional, View-Modus siehe Abschnitt 6:
# APP_DATA_SOURCE=view
# APP_VIEW_NAME=option_data
# Optional, nur für Demos mit passender Endpunktdatei, siehe Abschnitt 7:
# APP_DEMO_DATA=true
# Optional, siehe Abschnitt 8:
# APP_AUTH_BEARER_TOKEN='<Token>'
# Erforderlich, exakte verfügbare Version, siehe Abschnitt 11:
IMAGE_VERSION=0.0.1
# Optional, Native Image statt JVM:
# IMAGE_VARIANT=-native
```

Werte mit `$`, `#` oder Leerzeichen müssen in `.env` in einfachen Anführungszeichen stehen, damit Compose sie unverändert übernimmt; einfache Anführungszeichen innerhalb dieser Werte maskieren Sie mit `\'`. Kurzes Beispiel mit Platzhaltern: `DB_PASSWORD='<Passwortteil>$<Passwortteil> #<Zusatz>'`. Ersetzen Sie den gesamten Platzhalterwert durch Ihr eigenes Passwort und behalten Sie die äußeren einfachen Anführungszeichen bei.

Für Oracle verwenden Sie beispielsweise `DB_JDBC_URL=jdbc:oracle:thin:@//datenbank:1521/FREEPDB1`. Beachten Sie die Hinweise zur Erreichbarkeit des Datenbankhosts in Abschnitt 3. `ENDPOINT_CONFIG_FILE` bezeichnet eine vorhandene Datei auf dem Docker-Host; Compose bindet sie schreibgeschützt als Endpunktkonfiguration ein. Schützen Sie `.env` vor fremdem Zugriff:

```bash
chmod 600 .env
```

Führen Sie die Befehle im Betriebsverzeichnis aus. Wählen Sie genau ein Profil passend zu Ihrer Datenbank, da beide denselben lokalen Port verwenden:

```bash
docker compose --profile postgresql up -d
```

Für Oracle lautet der Startbefehl `docker compose --profile oracle up -d`. Der Dienst ist lokal unter `127.0.0.1:8080` erreichbar. Für den Zugriff aus epilot richten Sie einen HTTPS-Reverse-Proxy auf dem Docker-Host mit `127.0.0.1:8080` als Ziel ein. Der Hinweis aus Abschnitt 4 zu einem gemeinsamen Docker-Netzwerk für einen separaten Proxycontainer gilt nicht für diese Compose-Datei.

Nach Änderungen an `endpoints.yaml` erstellen Sie den Container mit `docker compose --profile <Profil> up -d --force-recreate` neu, wobei Sie `<Profil>` durch `postgresql` oder `oracle` ersetzen, damit auch ein atomarer Austausch der Datei wirksam wird.

Prüfen Sie die Startmeldungen und die Bereitschaft:

```bash
docker compose --profile postgresql logs --tail=100
curl --fail http://127.0.0.1:8080/q/health/ready
```

Verwenden Sie für Oracle bei Compose-Befehlen das Profil `oracle`. Die weiteren Betriebsprüfungen einschließlich der API-Aufrufe unter `/api/v1` stehen in Abschnitt 9, das Laden der Daten in Abschnitt 7. Zum Stoppen und Entfernen des Dienstcontainers:

```bash
docker compose --profile postgresql down
```

Die externe Datenbank und Ihre Konfigurationsdateien bleiben erhalten.

## 6. Kundendaten ausschließlich aus einer View lesen

Setzen Sie in `backend.env` beziehungsweise bei Compose in `.env`:

```dotenv
APP_DATA_SOURCE=view
APP_VIEW_NAME=option_data
```

`APP_DATA_SOURCE` akzeptiert genau `tables` (Vorgabe) oder `view`. Andere Werte verhindern den Start mit einer Fehlermeldung. `APP_VIEW_NAME` hat die Vorgabe `option_data` und wird im Tabellenmodus ignoriert. Ein Wechsel benötigt einen Neustart beziehungsweise das Neuerstellen des Containers. Er benötigt keinen Neubau und keine zusätzliche Imagevariante: PostgreSQL verwendet weiterhin `-postgresql` mit dem PostgreSQL JDBC Treiber und `DB_KIND=postgresql`, Oracle verwendet `-oracle` mit dem Oracle JDBC Treiber und `DB_KIND=oracle`. Die Datenbankart ist beim Bau festgelegt und muss zum Image passen. Es gelten weiterhin PostgreSQL ab 15 und Oracle ab 23ai 23.6.

Die Endpunkte kommen weiterhin aus `endpoints.yaml` gemäß Abschnitt 4. `QUARKUS_CONFIG_LOCATIONS` verweist auf die Datei im Container; `ENDPOINT_CONFIG_FILE` wählt bei Compose die Datei auf dem Host. Die Endpunktprüfung läuft auch im View-Modus. Nur konfigurierte Pfade sind erreichbar. Entfernte Pfade liefern nach dem Neustart HTTP 404.

Im View-Modus führt die Anwendung ausschließlich SELECT-Abfragen aus. Sie erstellt keine Tabellen oder Views und schreibt keine Zeilen. Liquibase ist einschließlich Migration und Bereinigung beim Start abgeschaltet, auch wenn diese separat aktiviert wurden. Hibernate behält `schema-management.strategy=none`. Es gibt keine Synchronisation nach `option_endpoint`; die Tabellen `option_endpoint`, `option_item` und Liquibase-Verwaltungstabellen werden nicht benötigt.

Der Kunde erstellt die View vor dem Start mit einem separaten Eigentümerkonto. Der Anwendungsbenutzer benötigt nur Anmeldung und SELECT auf die View, unter PostgreSQL zusätzlich USAGE auf deren Schema. Unter Oracle ist CREATE SESSION zur Anmeldung erforderlich. Er benötigt weder CREATE TABLE oder CREATE VIEW noch ALTER, INSERT, UPDATE, DELETE oder ein Tablespace-Kontingent. Der Eigentümer stellt die nötigen Rechte auf den zugrunde liegenden Objekten bereit; bei einer View mit Aufruferrechten muss er auch passende Leserechte für den Anwendungsbenutzer vergeben.

### Spaltenvertrag

Verwenden Sie unquotierte Bezeichner. PostgreSQL speichert die folgenden Namen klein, Oracle groß. `item_key` wird also in PostgreSQL zu `item_key` und in Oracle zu `ITEM_KEY`. Quoting und gemischte Schreibweise bei der Definition werden nicht unterstützt. Der View-Name darf optional ein Schema enthalten, etwa `kundendaten.option_data`. Jeder Teil beginnt mit einem ASCII-Buchstaben und enthält höchstens 63 ASCII-Buchstaben, Ziffern oder Unterstriche. Leerzeichen, Anführungszeichen und weitere Punkte sind nicht erlaubt. Ohne Schema gelten der PostgreSQL Suchpfad beziehungsweise das aktuelle Oracle Schema. Oracle Synonyme und materialisierte Views werden nicht unterstützt.

| Spalte | PostgreSQL | Oracle | Bedeutung |
|---|---|---|---|
| `endpoint_path` | `varchar(255)` | `VARCHAR2(255 CHAR)` | Konfigurierter Pfad ohne API-Basispfad, z. B. `tarife` |
| `item_key` | `varchar(255)` | `VARCHAR2(255 CHAR)` | JSON `key`, stabil und innerhalb des Endpunkts über alle Ebenen eindeutig |
| `label` | `varchar(500)` | `VARCHAR2(500 CHAR)` | JSON `label` |
| `item_value` | `varchar(500)` | `VARCHAR2(500 CHAR)` | JSON `value`, eindeutig je Endpunkt und Elternknoten |
| `info` | `text` | `CLOB` | Optionales JSON `info`; bei NULL fehlt das Feld |
| `sort_order` | `integer` | `NUMBER(10,0)` | Aufsteigende Reihenfolge, ganze Zahl im Bereich eines vorzeichenbehafteten 32-Bit-Integer |
| `item_level` | `smallint` | `NUMBER(1,0)` | Ebene 1, 2 oder 3; Dropdowns nur Ebene 1 |
| `parent_key` | `varchar(255)` | `VARCHAR2(255 CHAR)` | NULL auf Ebene 1, sonst `item_key` des Elternknotens im selben Endpunkt |
| `active` | `smallint` | `NUMBER(1,0)` | 1 für aktiv, 0 für inaktiv; kein BOOLEAN |

Alle Felder außer `info` und dem `parent_key` der Ebene 1 müssen befüllt sein. Pflichttexte dürfen nicht leer sein; Oracle behandelt leere Zeichenketten als NULL. Jeder Kindknoten liegt genau eine Ebene unter seinem Elternknoten. Verwenden Sie bei wiederholten Kundenschlüsseln einen zusammengesetzten, stabilen `item_key` und denselben Wert als `parent_key` der Kinder. Tabellen-IDs der Anwendung werden nicht verwendet. Zusätzliche Spalten sind erlaubt und werden ignoriert.

Die Anwendung prüft beim Start, dass das Objekt eine zugängliche View ist, alle neun Spalten unter ihren unquotierten Namen existieren und ihre JDBC Typen kompatibel sind. Fehlende Views, fehlende Spalten und inkompatible Typen verhindern den Start mit einer Meldung zu `APP_VIEW_NAME`. Die Prüfung liest keine Nutzdaten. Die korrekten Werte, Pflichtfelder, Eindeutigkeiten und Elternbeziehungen stellt der Kunde in seiner View sicher; die Anwendung legt dafür keine Constraints an.

Die API bleibt unverändert: Dropdowns akzeptieren keine Queryparameter, Kaskaden verwenden `level1` und `level2` mit den jeweiligen `item_value`-Werten. Die Anwendung löst diese in stabile Elternschlüssel auf und berücksichtigt nur aktive Knoten der passenden Ebene. Pro Antwort werden höchstens 1000 Einträge geliefert. Sortiert wird nach `sort_order`, dann `label`, dann `item_key`. Der eindeutige Schlüssel ist der stabile letzte Vergleichswert. Textreihenfolge folgt der Datenbankkollation; für eine datenbankübergreifend identische fachliche Reihenfolge verwenden Sie unterschiedliche `sort_order`-Werte. View-Datenänderungen benötigen keinen Neustart. Die Health Endpunkte `/q/health`, `/q/health/live` und `/q/health/ready` bleiben verfügbar.

### PostgreSQL Beispiel

Dieses eigenständige Beispiel erzeugt eine View mit einem E2E-TEST-Dropdown. Führen Sie es als View-Eigentümer aus. Für den produktiven Einsatz ersetzen Sie die Literale durch eine SELECT-Abfrage Ihrer bestehenden Daten mit denselben Aliasnamen und Typen.

```sql
CREATE VIEW option_data AS
SELECT
    CAST('tarife' AS varchar(255)) AS endpoint_path,
    CAST('E2E-TEST-basis' AS varchar(255)) AS item_key,
    CAST('E2E-TEST Basis' AS varchar(500)) AS label,
    CAST('E2E-TEST-BASIS' AS varchar(500)) AS item_value,
    CAST(NULL AS text) AS info,
    CAST(1 AS integer) AS sort_order,
    CAST(1 AS smallint) AS item_level,
    CAST(NULL AS varchar(255)) AS parent_key,
    CAST(1 AS smallint) AS active;

GRANT USAGE ON SCHEMA public TO epilot_reader;
GRANT SELECT ON option_data TO epilot_reader;
```

### Oracle Beispiel

Die View wird im Schema des Eigentümers erzeugt. Setzen Sie bei einem separaten Anwendungsbenutzer `APP_VIEW_NAME=kundendaten.option_data`, wenn das Eigentümerschema `kundendaten` heißt. Der Benutzer `epilot_reader` muss bereits existieren und CREATE SESSION besitzen.

```sql
CREATE VIEW option_data AS
SELECT
    CAST('tarife' AS VARCHAR2(255 CHAR)) AS endpoint_path,
    CAST('E2E-TEST-basis' AS VARCHAR2(255 CHAR)) AS item_key,
    CAST('E2E-TEST Basis' AS VARCHAR2(500 CHAR)) AS label,
    CAST('E2E-TEST-BASIS' AS VARCHAR2(500 CHAR)) AS item_value,
    TO_CLOB(NULL) AS info,
    CAST(1 AS NUMBER(10,0)) AS sort_order,
    CAST(1 AS NUMBER(1,0)) AS item_level,
    CAST(NULL AS VARCHAR2(255 CHAR)) AS parent_key,
    CAST(1 AS NUMBER(1,0)) AS active
FROM dual;

GRANT SELECT ON option_data TO epilot_reader;
```

Für eine Kaskade mit `endpoint_path=netz-produkt-tarif` liefern Sie beispielsweise einen Knoten `item_key=E2E-TEST-netz`, `item_value=E2E-TEST-NETZ`, `item_level=1`, `parent_key=NULL` und darunter `item_key=E2E-TEST-produkt`, `item_value=E2E-TEST-PRODUKT`, `item_level=2`, `parent_key=E2E-TEST-netz`. Ein dritter Knoten verwendet Ebene 3 und `parent_key=E2E-TEST-produkt`. Alle drei benötigen `label`, `sort_order` und `active=1`. Der Aufruf mit `level1=E2E-TEST-NETZ&level2=E2E-TEST-PRODUKT` liefert die dritte Ebene.

## 7. Daten im Tabellenmodus laden

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

### Homepage Demodaten optional laden

`APP_DEMO_DATA` ist standardmäßig `false`. Eine fehlende oder leere Variable bedeutet ebenfalls `false`. Erlaubt sind nur `true` und `false`, unabhängig von Großschreibung; umgebende Leerzeichen werden ignoriert. Ungültige Werte verhindern den Start. Verwenden Sie diese Funktion ausschließlich für Demos und lokale Erprobung in einer eigenen Datenbank, nicht für produktive Kundendaten. Sie funktioniert nur mit `APP_DATA_SOURCE=tables`. Zusammen mit `APP_DATA_SOURCE=view` verhindert `APP_DEMO_DATA=true` den Start, ohne Demodaten zu schreiben.

Die JSON Dateien sind im JVM und Native Image enthalten. Beim Start erfolgt kein Abruf von epilot-berater.de. Nach Liquibase und der Synchronisation der Endpunkte werden nur leere Endpunkte befüllt. Sobald für einen `endpoint_path` irgendeine Zeile existiert, bleiben alle vorhandenen Kundendaten und E2E-TEST Daten unverändert, auch inaktive Zeilen. Ein Neustart erzeugt keine Duplikate. Nicht konfigurierte Datenpakete werden mit einer Protokollmeldung übersprungen. Ohne einen passenden konfigurierten Endpunkt schlägt der Start fehl. Sind alle passenden Endpunkte bereits befüllt, startet die Anwendung ohne Einfügungen. Die Eltern werden vor ihren Kindern über `parent_id` eingefügt; der gesamte Ladevorgang erfolgt in einer Transaktion.

Kopieren Sie `config/endpoints.demo.yaml` aus dem Quellrepository als `endpoints.demo.yaml` in Ihr Betriebsverzeichnis. Die Datei enthält die Dropdowns `netzgebiete`, `tarife`, `ladeeinrichtungen`, `zaehlerarten`, `einspeisung` sowie `anliegen` mit drei Ebenen. Ohne Zugriff auf das Quellrepository erstellen Sie `endpoints.demo.yaml` mit folgendem Inhalt:

```yaml
app:
  endpoints:
    - path: netzgebiete
      type: dropdown
    - path: tarife
      type: dropdown
    - path: ladeeinrichtungen
      type: dropdown
    - path: zaehlerarten
      type: dropdown
    - path: einspeisung
      type: dropdown
    - path: anliegen
      type: cascade
      levels: 3
```

Das generische `endpoints.example.yaml` bleibt verfügbar. Für die Compose Datei aus Abschnitt 5 ergänzen beziehungsweise ersetzen Sie in Ihrer lokalen `.env`:

```dotenv
APP_DATA_SOURCE=tables
APP_DEMO_DATA=true
ENDPOINT_CONFIG_FILE=./endpoints.demo.yaml
```

Starten Sie mit `docker compose --profile postgresql up -d --force-recreate`, für Oracle mit `--profile oracle`. Die Compose Datei im Quellrepository unterstützt dieselben Variablen; dort lautet der Dateipfad `ENDPOINT_CONFIG_FILE=./config/endpoints.demo.yaml` und beim ersten Start ist zusätzlich `--build` erforderlich. Ohne diese ausdrückliche Aktivierung bleibt das bisherige Startverhalten erhalten. Für `docker run` setzen Sie die beiden APP Variablen in Ihrer lokalen `backend.env` und binden die Demo Endpunktdatei wie in Abschnitt 4 ein. Aktivieren Sie die Funktion nicht in CI oder bereitgestellten Umgebungen.

Die 828 Netzgebiete stammen aus dem Marktstammdatenregister (MaStR) unter Datenlizenz Deutschland Namensnennung Version 2.0. Die ursprüngliche Quellenangabe `quelle` bleibt in der mitgelieferten Datei `netzgebiete.json` erhalten.

Ausnahme in den Quelldaten: Zwei Netzgebiete haben denselben Code `10001568`. Damit alle 828 Einträge mit eindeutigen Werten erhalten bleiben, erhalten ausschließlich `SNB930329896650` (EVU Langenpreising) und `SNB994196581439` (Gemeindewerke Langenpreising) die Werte `10001568-SNB930329896650` beziehungsweise `10001568-SNB994196581439` als `item_value`. Bei allen anderen Einträgen bleibt `item_value` der unveränderte Code. Der Mapper setzt diese Ausnahme um; die mitgelieferte JSON Quelldatei bleibt unverändert.


## 8. Optionalen Zugriffsschutz aktivieren

Ergänzen Sie in `backend.env` beziehungsweise bei Compose in `.env` die Variable `APP_AUTH_BEARER_TOKEN` mit einem geheimen Token und erstellen Sie den Container mit derselben Startanweisung neu. Ohne Token oder bei leerem Wert ist die API öffentlich. Bei gesetztem Token verlangen API-Aufrufe den Header `Authorization: Bearer <Token>`. Der Dienst protokolliert den Token nicht. Die Health Endpunkte bleiben ohne Authentifizierung erreichbar.

## 9. Betrieb prüfen

```bash
curl --fail http://localhost:8080/q/health/ready
curl --fail http://localhost:8080/q/health/live
```

`/q/health` fasst die Prüfungen zusammen; `/q/health/started` prüft den Startzustand. Warten Sie beim ersten Start auf die Datenbankverbindung und im Tabellenmodus auf die erfolgreiche Migration. Im View-Modus muss stattdessen die View-Prüfung erfolgreich sein. Prüfen Sie anschließend `/api/v1/tarife` und die Kaskade mit Ihrem HTTP Werkzeug, bei aktiviertem Schutz mit dem Authorization Header. Ohne Einträge ist `[]` eine gültige Antwort.

Eine erfolgreiche Antwort enthält Objekte wie:

```json
[{"key":"basis","label":"Basis","value":"BASIS","info":"12 Monate Laufzeit"}]
```

HTTP 401 bedeutet fehlende oder falsche Authentifizierung. HTTP 404 bedeutet einen unbekannten oder inaktiven Endpunkt beziehungsweise einen nicht gefundenen Elternwert. HTTP 400 weist auf ungültige, doppelte oder unerwartete Parameter hin. Dropdowns erlauben keine Queryparameter. Bei Kaskaden muss `level1` vor `level2` angegeben werden; eine Kaskade mit zwei Ebenen erlaubt kein `level2`.

## 10. epilot Apps einrichten

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

## 11. Version gezielt aktualisieren

`IMAGE_VERSION` enthält weiterhin nur die Version, etwa `0.0.1`. Für Native setzen Sie zusätzlich `IMAGE_VARIANT=-native` in `.env`. Compose setzt daraus zum Beispiel `0.0.1-postgresql-native` zusammen. Eine fehlende oder leere Variable `IMAGE_VARIANT` wählt JVM; bestehende `.env` Dateien funktionieren unverändert. Zum Wechsel auf JVM entfernen Sie `IMAGE_VARIANT` auch aus der Shell-Umgebung. Behalten Sie die gewählte Buildart bei Updates bewusst bei und prüfen Sie die exakten Tags aus Abschnitt 2.

Sichern Sie die Datenbank und Ihre Konfiguration. Lesen Sie die Versionshinweise und testen Sie die neue Version mit einer separaten Datenbank in Ihrer Testumgebung. Laden Sie das Image mit der exakten neuen Versionsnummer und demselben Datenbanksuffix. Verwenden Sie für reproduzierbare Updates keinen `latest` Tag.

Bei Compose behalten Sie Ihre lokale `compose.yaml`. Prüfen Sie anhand der aktuellen Anleitung und der Versionshinweise, ob Anpassungen an der Compose Datei oder neue Variablen in `.env` erforderlich sind. Setzen Sie `IMAGE_VERSION` in `.env` auf die exakte neue Version ohne Datenbanksuffix. Entfernen Sie einen abweichenden Wert aus der Shell-Umgebung, da dieser Vorrang vor `.env` hat. Führen Sie anschließend `docker compose --profile postgresql pull` und `docker compose --profile postgresql up -d` aus, für Oracle jeweils mit `--profile oracle`.

Bei Verwendung von `docker run` stoppen und entfernen Sie nach dem Laden des neuen Images den bisherigen Container und führen Sie den Startbefehl aus Abschnitt 4 mit der neuen Versionsnummer aus. Bewahren Sie die externe Datenbank und Konfigurationsdateien auf. Prüfen Sie Health Endpunkte und repräsentative Dropdowns und Kaskaden. Ein Zurücksetzen des Images setzt bei inkompatiblen Datenbankmigrationen auch das Wiederherstellen einer passenden Sicherung voraus.

Diese Anleitung wird bei jeder externen Veröffentlichung automatisch im Kundenrepository unter `epilot-dynamic-backend/README.md` aktualisiert. Sie ist das einzige veröffentlichte Kundendokument neben dem Containerimage und enthält die Compose Beispiele sowie den automatisch ergänzten Abschnitt `Verfügbare Tags`.
