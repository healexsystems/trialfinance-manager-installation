# TrialFinance Manager (TFM)

Der Healex TrialFinance Manager verbessert die Abrechnung klinischer Studien durch erhöhte Transparenz und Effizienz. Kliniken erhalten eine klare Übersicht darüber, welche Leistungen zu welchem Zeitpunkt und in welcher Weise abgerechnet wurden. Externe Leistungserbringer können ihre Leistungen quittieren, was die Nachverfolgung und Abrechnung erleichtert. Das System ermöglicht zudem automatisierte Rechnungsstellungen, sobald Studienkoordinatoren die relevanten Posten freigeben. Dies steigert die Produktivität der Kliniken und stellt sicher, dass alle erbrachten Leistungen transparent und korrekt in Rechnung gestellt werden. Der Healex TrialFinance Manager ist DSGVO-konform und bietet ein abgestimmtes Rollen- und Rechte-Management sowie einen Freigabeprozess für Rechnungen.

- [Systemanforderungen](#systemanforderungen)
    * [Umgebungseinrichtung](#umgebungseinrichtung)
    * [Docker Installation](#docker-installation)
    * [Lizensierung & Image Download](#lizensierung-und-download-des-docker-images)
    * [Hardwareanforderungen](#hardwareanforderungen)
    * [Clientseitige Anforderungen](#clientseitige-anforderungen)
- [Erste Schritte](#erste-Schritte)
- [Umgebungsvariablen](#umgebungsvariablen)
- [Docker Secrets](#docker-Secrets)
- [OIDC Konfiguration](#oidc-konfiguration)
  - [Einrichten der OIDC Clients in ClinicalSite](#einrichten-der-oidc-clients-in-clinicalsite)
  - [OIDC System](#oidc-system)
  - [Backend System](#backend-system)    
- [SSL Proxy Server](#ssl-proxy-server)
  - [Selbst signierte Zertifikate](#selbst-signierte-zertifikate)
- [Login](#login)
- [Container Shell](#zugriff-auf-die-container-shell)
- [Logs](#anzeige-von-container-logs)
- [Upgrade](#upgrade)

# Systemanforderungen
## Umgebungseinrichtung

Der TrialFinance Manager ist containerisiert und lässt sich ausschließlich mit einer OCI-kompatiblen Runtime betreiben. Der Betrieb, bspw. in Docker, ist problemlos möglich und wird aufgrund bisheriger Erfahrungen von Healex empfohlen.

## Docker Installation

Hierfür wird eine Docker Umgebung benötigt (https://www.docker.com/products/container-runtime). 
Mit dieser können die Hosts (unabhängig davon, ob real, virtuell, lokal oder remote) ausgeführt und angesteuert werden.

Installieren Sie hierfür die Docker Engine: https://docs.docker.com/get-docker

## Lizensierung und Download des Docker Images

Es wird ein Zugang zum privaten Healex Docker Repository benötigt.  
Wenden Sie sich hierfür an <support@healex.systems>, um die Lizenzbedingungen zu besprechen. Sobald die Lizenz eingerichtet ist, erhalten Sie ihren Kontonamen, mit welchem der Zugriff zum Docker Repository ermöglicht wird.

## Hardwareanforderungen

| Service                    | vCPU    | RAM    | Festplattenspeicher |
|----------------------------|---------|--------|---------------------|
| TrialFinance Manager       | 2       | 4 GB   | 5 GB                |

Diese Mindestwerte eignen sich für Tests, Prototypen, Pre-PROD und erfahrungsgemäß für den anfänglichen Betrieb der Produktionsumgebung. 
Abhängig von der Entwicklung der realen Nutzung können sich hiervon abweichende Systemanforderungen ergeben. 
Die Hardwareanforderungen sollten vom hauseigenen Betrieb überwacht und angepasst werden.

## Clientseitige Anforderungen

  - Keine speziellen Hardwareanforderungen notwendig
  - Aktivierung von JavaScript und Cookies (i.d.R. Browser Standardeinstellungen)
  - Standardkonformer Browser: aktuelle Versionen (nicht älter als 1 Jahr) des
      * Google Chrome
      * Mozilla Firefox
      * Microsoft Edge
      * Safari


# Erste Schritte   
Am einfachsten lässt sich der TFM über `docker compose` starten. Hierfür werden folgende Services benötigt:

  - frontend: Frontend Container des TrialFinance Managers
  - backend: Backend Container des TrialFinance Managers
  - db: Postgres Datenbank Container

Beispiel für eine compose.yml:

```yaml 
version: '3.8'

volumes:
  db:

services:
  frontend:
    image: healexsystems/sf-frontend:2815
    container_name: tfm-frontend    
    read_only: true
    depends_on:
      - backend    
    environment:
      BACKEND_PUBLIC_BASE_URL: http://localhost:4000
      CLINICALSITE_BASE_URL: https://clinicalsite.example.com
      APP_CS_OIDC_SCOPE: openid+profile+email
      APP_CS_CLIENT_ID: client1
      AI_BACKEND_BASE_URL: https://backend.ai-visitplan.example.com
    ports:
      - 8080:80
    volumes:
      - type: tmpfs
        target: /usr/share/nginx/html/tmpfs
        tmpfs:
          size: 1000
          mode: 0777
      - type: tmpfs
        target: /tmp

  backend:
    image: healexsystems/sf-backend:2815
    container_name: tfm-backend
    depends_on:
      - db
    environment:
      FRONTEND_URL: http://localhost:8080
      DB_HOST: db
      DB_PORT: 3306
      DB_NAME: exmple-database
      DB_USER: example-user
      DB_PASS: secret_pw
      DB_OWNER_USER: migration
      DB_OWNER_PASS: migration_secret_pw      
      CLINICALSITE_BASE_URL: https://clinicalsite.example.com
      CLINICALSITE_ROLES_REPORT_PATH: /api/query/tfm_roles
      OIDC_CLIENT_ID: client2
      OIDC_CLIENT_SECRET: oidc_client_secret
      AI_BACKEND_BEARER_TOKEN: long_random_bearer_token_goes_here
    ports:
      - 4000:80      
    volumes:
      - type: bind
        source: ./licence.key
        target: /app/licence.key
        read_only: true    
      - type: bind
        source: ./public.pem
        target: /app/public.pem
        read_only: true

  db:
    image: postgres:15-alpine
    container_name: tfm-db
    environment:
      POSTGRES_PASSWORD: secret-root-pw
    volumes:
      - db:/var/lib/postgresql/data    
      - type: bind
        source: ./init.sql
        target: /docker-entrypoint-initdb.d/init.sql
        read_only: true
```

# Umgebungsvariablen

Beim Starten der Images ist es möglich, die Initialisierung der Front- und Backend-Instanzen mittels folgender Umgebungsvariablen anzupassen:

Frontend:

| Umgebungsvariable          | Erforderlich | Kommentar/Beschreibung                                                     | Standart Wert |  Beispiel                               |
|----------------------------|--------------|----------------------------------------------------------------------------|---------------|-----------------------------------------|
| CLINICALSITE_BASE_URL      |       Ja     | Base URL der ClinialSite Instanz                                           |               | https://clinicalsite.example.com        |
| APP_CS_CLIENT_ID           |       Ja     | Name des Browser-basierten ClinicalSite OIDC Clients (Authentifizierung ohne Kennwort) |   | client1                                 |
| APP_CS_OIDC_SCOPE          |       Ja     | Scope des ClinicalSite OIDC Clients                                        |               | openid+profile+email                    |
| BACKEND_PUBLIC_BASE_URL    |       Ja     | URL der Backend Instanz, diese muss über den Browser erreichbar sein       |               | https://backend.tfm.example.com         |
| AI_BACKEND_BASE_URL        |       Nein   | Backend URL der AI-Instanz. Variable muss leer sein bzw. nicht gesetzt werden, um sicherustellen, dass der KI-Import deaktviert ist. || https://backend.ai-visitplan.example.com                              |

Backend:

| Umgebungsvariable          | Erforderlich | Kommentar/Beschreibung                                                 | Standard Wert |  Beispiel                              |
|----------------------------|--------------|------------------------------------------------------------------------|---------------|----------------------------------------|
| FRONTEND_URL               |       Ja     | URL der Frontend Instanz                                               |               | https://tfm.example.com                |
| DB_HOST                    |       Ja     | URL oder IP der Datenbank                                              |               | 10.8.0.6                               |
| DB_PORT                    |       Ja     | Datenbank Port                                                         |               | 3306                                   |
| DB_NAME                    |       Ja     | Datenbank Name                                                         |               | tfm-t                                  |
| DB_USER                    |       Ja     | Datenbank Benutzer                                                     |               | tfm                                    |
| DB_PASS                    |       Ja     | Passwort des Datenbank-Benutzers                                       |               |                                        |
| DB_OWNER_USER              |       Ja     | Datenbank Migrations-Benutzer                                          |               | migration                              |
| DB_OWNER_PASS              |       Ja     | Passwort des Datenbank-Migrations-Benutzers                            |               |                                        |
| CLINICALSITE_BASE_URL      |       Ja     | Base URL der ClinialSite Instanz                                       |               | https://clinicalsite.example.com       |
| CLINICALSITE_ROLES_REPORT_PATH |   Ja     | Pfad zum ClinicalSite Report, in welchem die Benutzer Rollen und Rechte defniert sind. Der API-Bezeichner des Reports ist in ClinicalSite frei wählbar und muss im Pfad korrekt angegeben werden. | | "/api/query/${REPORT_NAME}", <br>z.B. "/api/query/tfm_roles"
| OIDC_CLIENT_ID             |       Ja     | Name des Server-basierten ClinicalSite OIDC Clients (Authentifizierung per Kennwort)|  | client2                                |
| OIDC_CLIENT_SECRET         |       Ja     | Kennwort des ClinicalSite OIDC Clients                                 |               |                                        |
| AI_BACKEND_BEARER_TOKEN    |       Nein   | Bearer Token für den AI-Service. Diese muss mit dem Bearer Token des AI-Services übereinstimmen.                                        |               |                                        |

# Docker Secrets
Als eine Alternative zur Weitergabe vertraulicher Informationen über Umgebungsvariablen können Docker-Secrets verwendet werden.
Um eine Umgebungsvariable als Docker Secret zu verwenden, muss `_FILE` an den `$VARIABLEN_NAMEN` angehängt werden und der Pfad zu
der Secret-Datei festgelegt werden (Standard Pfad ist `/run/secrets`)

Beispiel 1: Wert wird über den Environment-Abschnitt gesetzt:

```yaml 
 services:
   backend:
     image: healexsystems/sf-backend:latest
     environment:
       - LICENCE_SECRET=secret-pw
```

Beispiel 2: Wert wird über ein Docker-Secret gesetzt:

```yaml 
 services:
   backend:
     image: image: healexsystems/sf-backend:latest
     environment:
       - LICENCE_SECRET_FILE=/run/secrets/app-secret
     secrets:
       - app-secret

 secrets:
   app-secret:
     file: secret.txt
```

Umgebungsvariablen, welche nach Bedarf mittels Docker-Secrets eingebunden werden können:

Backend:

* `DB_HOST`
* `DB_PORT`
* `DB_NAME`
* `DB_USER`
* `DB_PASS`
* `DB_OWNER_USER`
* `DB_OWNER_PASS`
* `OIDC_CLIENT_ID`
* `OIDC_CLIENT_SECRET`
* `AI_BACKEND_BEARER_TOKEN`

# OIDC Konfiguration

## Einrichten der OIDC Clients in ClinicalSite

Im ClinicalSite werden zwei unterschiedliche externe Systeme benötigt:
* OIDC
* Backend

Hierfür werden administrative Rechte in ClinicalSite benötigt. Navigieren Sie zu: <br> 
* ` Verwaltung - Externe Systeme`
* Erstellen Sie ein `Neues externes System`

## OIDC System
* Wählen Sie eine beliebige Bezeichnung für den Client im Feld `Name`
* Wählen Sie einen Namen für den `Bezeichner (Client ID)`. Dieser Wert muss im Frontend-Service für die Umgebungsvariable `APP_CS_CLIENT_ID` gesetzt werden.
* Wählen Sie Client-Typ: `Browser-basiert (Authentifizierung ohne Kennwort, PKCE-Unterstützung erforderlich)`
* Tragen Sie folgende `Redirect-URL` ein, z.B.:
  * https://trialfinance.example.com/
  * Achten Sie darauf, dass die URL mit `/`endet
* Legen Sie den Client an

Das Kennwort (Client Secret) wird nicht benötigt und wird deshalb nicht in TrialFinance hinterlegt.

## Backend System
* Wählen Sie eine beliebige Bezeichnung für den Client im Feld `Name`
* Wählen Sie einen Namen für den `Bezeichner (Client ID)`. Dieser Wert muss im Backend-Service für die Umgebungsvariable `OIDC_CLIENT_ID` gesetzt werden.
* Wählen Sie Client-Typ: `Server-basiert (Authentifizierung per Kennwort)`
* Legen Sie den Client an
* Das Kennwort (Client Secret) muss im Backend-Service für die Umgebungsvariable `OIDC_CLIENT_SECRET` gesetzt werden.


# SSL Proxy-Server
Eine SSL-Verschlüsselung mittels eines Proxy Servers ist empfohlen und im produktiven Betrieb zwingend.

## Selbst signierte Zertifikate
Für den Fall, dass TrialFinance mit Servern kommuniziert, welche selbst signierte Zertifikate verwenden, muss das Root Zertifikat der eigenen Zertifizierungsstelle dem Container hinzugefügt werden.
Dies ist beispielsweise der Fall, wenn TrialFinance via OIDC mit ClinicalSite angebunden wird und ClinicalSite selbst signierte Zertifikate verwendet.

Das Einbinden des Root Zertifikats geschieht über das Setzen der folgenden Umgebungsvariable:
```yaml
    environment:
      NODE_EXTRA_CA_CERTS=/app/rootCA.crt
```

Hierfür muss das Root Zertifikat ebenfalls über den Volume-Abschnitt in den Container gemounted werden.

```yaml
    volumes:
      - type: bind
        source: ./rootCA.crt
        target: /app/rootCA.crt
        read_only: true   
```

# Login
Ein Login ist ausschließlich über OIDC möglich.

# Zugriff auf die Container-Shell
Mit dem Befehl `docker exec` können Befehle innerhalb des laufenden Containers ausgeführt werden. Mit der folgenden Befehlszeile wird eine Shell im Front- oder Backend Container geöffnet.

```shell
docker exec -it docker Container-ID /bin/bash
```

# Anzeige von Container Logs
```shell
docker logs Container-ID
```

# Upgrade
* Seit v2794: ENV "LICENCE_SECRET" entfällt. Wird noch supported, sollte allerdings durch das neue Verschlüsselungsverfahren ersetzt werden. Hierfür muss die Datei "public.pem" über den Volume Abschnitt gesetzt werden (empfohlen)

* Seit v2777: ENV `AI_SERVICE_FRONTEND_URL` und `AI_SERVICE_BACKEND_URL` wird durch `AI_BACKEND_BASE_URL` ersetzt. <br> 
  Backend Umgebungsvariable `AI_BACKEND_BEARER_TOKEN` hinzugefügt 


