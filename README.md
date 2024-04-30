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
- [SSL Proxy Server](#ssl-proxy-server)
- [Login](#login)
- [Container Shell](#zugriff-auf-die-container-shell)
- [Logs](#anzeige-von-container-logs)

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
  - db: MariaDB Datenbank Container

Beispiel für eine compose.yml:

```yaml 
version: '3.8'

volumes:
  db:

services:
  frontend:
    image: healexsystems/sf-frontend:2507
    container_name: tfm-frontend    
    read_only: true
    depends_on:
      - backend    
    environment:
      BACKEND_PUBLIC_BASE_URL: http://localhost:4000
      CLINICALSITE_BASE_URL: https://clinicalsite.example.com
      APP_CS_OIDC_SCOPE: openid+profile+email
      APP_CS_CLIENT_ID: client1
      AI_SERVICE_BASE_URL: https://ai-visitplan.tfm.example.com
      AI_SERVICE_BACKEND_URL: https://backend.ai-visitplan.tfm.example.com
      AI_SERVICE_BACKEND_API_URL: /api/code/
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
    image: healexsystems/sf-backend:latest
    container_name: tfm-backend
    depends_on:
      - db
    environment:
      FRONTEND_URL: http://localhost:8080
      DB_HOST: db
      DB_PORT: 3306
      DB_NAME: exmple-database
      DB_USER: example-user
      DB_PASS: secret-pw
      DB_OWNER_USER: migration
      DB_OWNER_PASS: migration_secret_pw      
      CLINICALSITE_BASE_URL: https://clinicalsite.example.com
      OIDC_CLIENT_ID: client2
      OIDC_CLIENT_SECRET: oidc_client_secret
      LICENCE_SECRET: license_secret
      CSRF_SECRET: csrf_secret_minimum_32_characters
    ports:
      - 4000:80      
    volumes:
      - ./licence.key:/app/licence.key:ro

  db:
    image: postgres:15-bookworm
    container_name: tfm-db
    environment:
      POSTGRES_PASSWORD: secret-root-pw
    volumes:
      - db:/var/lib/postgresql/data    
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
```

# Umgebungsvariablen

Beim Starten der Images ist es möglich, die Initialisierung der Front- und Backend-Instanzen mittels folgender Umgebungsvariablen anzupassen:

Frontend:

| Umgebungsvariable          | Erforderlich | Kommentar/Beschreibung                                                     | Standart Wert |  Beispiel                               |
|----------------------------|--------------|----------------------------------------------------------------------------|---------------|-----------------------------------------|
| CLINICALSITE_BASE_URL      |       Ja     | Base URL der ClinialSite Instanz                                           |               | https://clinicalsite.example.com        |
| APP_CS_CLIENT_ID           |       Ja     | Name des Browser-basierten ClinicalSite OIDC Clients (Authentifizierung ohne Kennwort) |               | client1                                 |
| APP_CS_OIDC_SCOPE          |       Ja     | Scope des ClinicalSite OIDC Clients                                        |               | openid+profile+email                    |
| BACKEND_PUBLIC_BASE_URL    |       Ja     | URL der Backend Instanz, diese muss über den Browser erreichbar sein       |               | https://backend.tfm.example.com         |
| AI_SERVICE_BASE_URL        |       Ja     | Base URL der AI-Instanz. Nur als Platzhalter erforderlich, da Feature noch in Entwicklung ist  |               | https://ai-visitplan.tfm.example.com    |
| AI_SERVICE_BACKEND_URL     |       Ja     | Backend URL der AI-Instanz. Nur als Platzhalter erforderlich, da Feature noch in Entwicklung ist |               | https://be.ai-visitplan.tfm.example.com |
| AI_SERVICE_BACKEND_API_URL |       Ja     | Api Pfad der AI-Instanz. Nur als Platzhalter erforderlich, da Feature noch in Entwicklung ist |               | /api/code/                              |

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
| OIDC_CLIENT_ID             |       Ja     | Name des Server-basierten ClinicalSite OIDC Clients (Authentifizierung per Kennwort)|               | client2                   |
| OIDC_CLIENT_SECRET         |       Ja     | Kennwort des ClinicalSite OIDC Clients                                 |               |                                        |
| LICENCE_SECRET             |       Ja     | Entschlüsselungs-Kennwort für die Lizenz Signatur                      |               |                                        |
| CSRF_SECRET                |       Ja     | Kennwort zur Verhinderung von Cross-Site Request Forgery Attacken, Mindestlänge: 32 Zeichen     |               |              |

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
       - CSRF_SECRET=secret-pw
```

Beispiel 2: Wert wird über ein Docker-Secret gesetzt:

```yaml 
 services:
   backend:
     image: image: healexsystems/sf-backend:latest
     environment:
       - CSRF_SECRET_FILE=/run/secrets/app-secret
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
* `LICENCE_SECRET`
* `CSRF_SECRET`

# SSL Proxy-Server
Eine SSL-Verschlüsselung mittels eines Proxy Servers ist empfohlen und im produktiven Betrieb zwingend.

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