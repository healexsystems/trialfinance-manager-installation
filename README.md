# TrialFinance Manager (TFM)

Der Healex TrialFinance Manager verbessert die Abrechnung klinischer Studien durch erhöhte Transparenz und Effizienz. Kliniken erhalten eine klare Übersicht darüber, welche Leistungen zu welchem Zeitpunkt und in welcher Weise abgerechnet wurden. Externe Leistungserbringer können ihre Leistungen quittieren, was die Nachverfolgung und Abrechnung erleichtert. Das System ermöglicht zudem automatisierte Rechnungsstellungen, sobald Studienkoordinatoren die relevanten Posten freigeben. Dies steigert die Produktivität der Kliniken und stellt sicher, dass alle erbrachten Leistungen transparent und korrekt in Rechnung gestellt werden. Der Healex TrialFinance Manager ist DSGVO-konform und bietet ein abgestimmtes Rollen- und Rechte-Management sowie einen Freigabeprozess für Rechnungen.

- [Systemanforderungen](#systemanforderungen)
    * [Umgebungseinrichtung](#umgebungseinrichtung)
    * [Docker Installation](#docker-installation)
    * [Lizenzinformationen für TrialFinance](#lizenzinformationen-für-trialfinance)
    * [Download des Docker Images](#download-des-docker-images)
    * [Hardwareanforderungen](#hardwareanforderungen)
    * [Clientseitige Anforderungen](#clientseitige-anforderungen)
- [Erste Schritte](#erste-Schritte)
- [Umgebungsvariablen](#umgebungsvariablen)
- [Docker Secrets](#docker-Secrets)
- [ClinicalSite Konfiguration](#clinicalsite-konfiguration)
  - [Erstellung der OIDC Clients](#erstellung-der-oidc-clients)
    - [OIDC System](#oidc-system)
    - [Backend System](#backend-system)
  - [Erstellung des Rollen- und Rechte-Reports](#erstellung-des-rollen--und-rechte-reports)
- [SSL Proxy Server](#ssl-proxy-server)
  - [Selbst signierte Zertifikate](#selbst-signierte-zertifikate)
- [Anmeldung](#anmeldung)
- [Zugriff auf die Container-Shell](#zugriff-auf-die-container-shell)
- [Anzeige von Container-Logs](#anzeige-von-container-logs)
- [Upgrade Notes](#upgrade-notes)

# Systemanforderungen
## Umgebungseinrichtung

Der TrialFinance Manager ist containerisiert und lässt sich ausschließlich mit einer OCI-kompatiblen Runtime betreiben. Der Betrieb, bspw. in Docker, ist problemlos möglich und wird aufgrund bisheriger Erfahrungen von Healex empfohlen.

## Docker Installation

Hierfür wird eine Docker Umgebung benötigt (https://www.docker.com/products/container-runtime). 
Mit dieser können die Hosts (unabhängig davon, ob real, virtuell, lokal oder remote) ausgeführt und angesteuert werden.

Installieren Sie hierfür die Docker Engine: https://docs.docker.com/get-docker

## Lizenzinformationen für TrialFinance

Um eine Lizenz zu erhalten, wenden Sie sich bitte an <support@healex.systems>. Die Lizenz umfasst zwei Dateien:

- Lizenzschlüssel (*.key-Datei)
- Öffentlicher Schlüssel (*.pem-Datei)

Diese Dateien müssen per Volume-Bind-Mount in den Backend-Container von TrialFinance eingebunden werden:

```yaml 
    volumes:
      - type: bind
        source: ./licence.key
        target: /app/licence.key
        read_only: true    
      - type: bind
        source: ./public.pem
        target: /app/public.pem
        read_only: true
```

Stellen Sie sicher, dass die Berechtigungen der Lizenzdateien auf 644 (rw-r--r--) gesetzt sind:

- 6 für den Besitzer: Lesen (r) + Schreiben (w) → 4 + 2 = 6
- 4 für die Gruppe: Lesen (r)
- 4 für alle anderen: Lesen (r)

## Download des Docker Images

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
volumes:
  db:

services:
  frontend:
    image: healexsystems/sf-frontend:3217
    container_name: tfm-frontend    
    read_only: true
    depends_on:
      - backend    
    environment:
      BACKEND_PUBLIC_BASE_URL: http://localhost:4000
      CLINICALSITE_BASE_URL: https://clinicalsite.example.com
      APP_CS_OIDC_SCOPE: openid+profile+email
      APP_CS_CLIENT_ID: client1
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
    image: healexsystems/sf-backend:3217
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
      AI_BACKEND_BASE_URL: https://backend.ai-visitplan.example.com
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
| AI_BACKEND_BASE_URL        |       Nein   | Backend URL der AI-Instanz. Variable muss leer sein bzw. nicht gesetzt werden, um sicherustellen, dass der KI-Import deaktviert ist. || https://backend.ai-visitplan.example.com                              |

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
       - OIDC_CLIENT_SECRET=secret-pw
```

Beispiel 2: Wert wird über ein Docker-Secret gesetzt:

```yaml 
 services:
   backend:
     image: image: healexsystems/sf-backend:latest
     environment:
       - OIDC_CLIENT_SECRET_FILE=/run/secrets/app-secret
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

# Clinicalsite Konfiguration

## Erstellung der OIDC Clients

Im ClinicalSite werden zwei unterschiedliche externe Systeme benötigt:
* OIDC
* Backend

Hierfür werden administrative Rechte in ClinicalSite benötigt. Navigieren Sie zu: <br> 
* ` Verwaltung - Externe Systeme`
* Erstellen Sie ein `Neues externes System`

### OIDC System
* Wählen Sie eine beliebige Bezeichnung für den Client im Feld `Name`
* Wählen Sie einen Namen für den `Bezeichner (Client ID)`. Dieser Wert muss im Frontend-Service für die Umgebungsvariable `APP_CS_CLIENT_ID` gesetzt werden.
* Wählen Sie Client-Typ: `Browser-basiert (Authentifizierung ohne Kennwort, PKCE-Unterstützung erforderlich)`
* Tragen Sie folgende `Redirect-URL` ein, z.B.:
  * https://trialfinance.example.com/
  * Achten Sie darauf, dass die URL mit `/`endet
* Legen Sie den Client an

Das Kennwort (Client Secret) wird nicht benötigt und wird deshalb nicht in TrialFinance hinterlegt.

### Backend System
* Wählen Sie eine beliebige Bezeichnung für den Client im Feld `Name`
* Wählen Sie einen Namen für den `Bezeichner (Client ID)`. Dieser Wert muss im Backend-Service für die Umgebungsvariable `OIDC_CLIENT_ID` gesetzt werden.
* Wählen Sie Client-Typ: `Server-basiert (Authentifizierung per Kennwort)`
* Legen Sie den Client an
* Das Kennwort (Client Secret) muss im Backend-Service für die [Umgebungsvariable](#umgebungsvariablen) `OIDC_CLIENT_SECRET` gesetzt werden.

## Erstellung des Rollen- und Rechte-Reports

Hierfür werden administrative Rechte in ClinicalSite benötigt. Navigieren Sie zu: <br> 
* `Verwaltung - Reports`
* Erstellen Sie einen `Neuen Report`

Im folgenden werden lediglich die notwendigen Parameter gelistet:

* Wählen Sie eine beliebige Bezeichnung für den Report im Feld `Name`
* Wählen Sie einen Namen für den `API-Bezeichner`. Der Name ist Teil der [Umgebungsvariable](#umgebungsvariablen) `CLINICALSITE_ROLES_REPORT_PATH` des Backend-Services
* Wählen Sie als `Externe Systeme mit Zugriff` den Client des [Backend Systems](#backend-system)
* Das Feld `SQL`enthält die folgende SQL-Abfrage, wobei die `report parameter`: `tenant`, `root_orgunit` und `manager_orgunit` entsprechend gesetzt werden müssen

<br>

```sql
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
-- report parameters                  -
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
SELECT
     1 AS tenant,
     10 AS root_orgunit,
     11 AS manager_orgunit;
```

<br>

| SQL-Parameter   | Kommentar/Beschreibung                                                                                                                                                                                                           | Beispiel |
|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------|
| tenant          | ID des Mandanten                                                                                                                                                                                                                | 1        |
| root_orgunit    | ID der übergeordneten Organisationseinheit                                                                                                                                                                                      | 10       |
| manager_orgunit | ID der Organisationseinheit, in der die Verwalter, Manager oder Koordinatoren für das Modul Finance hinterlegt sind (in der Regel trägt diese Organisationseinheit den Namen „TrialSite Finance Manager“). Die affiliierten Benutzer werden in der Applikation Finance als „clinic-user“ bezeichnet. Sie besitzen umfassende Rechte innerhalb der Anwendung und können alle Studien einsehen   | 11       |

<br>


Die entsprechenden IDs für Mandanten und Organisationseinheiten können dem URL-Pfad entnommen werden:

  - Mandanten: `https://clinicalsite.example.com/tenant/{ID}`
  - Organisationseinheiten: `https://clinicalsite.example.com/ou/{ID}`

<br>


```md
SQL Abfrage:
```

```sql
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
-- report parameters                                         -
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
CREATE TEMPORARY TABLE report_parameters AS
SELECT
     10 AS tenant,
     5524 AS  root_orgunit,
     5525 AS  manager_orgunit;

-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
-- org-unit relations                                        -
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
CREATE TEMPORARY TABLE orgunit_relations AS
SELECT
    tenant.id         AS tenant,
    tenant.name       AS tenant_name,
    orgunit.id        AS orgunit,
    orgunit.parent_ou AS parent_orgunit,
    orgunit.name      AS orgunit_name
FROM orgunit
INNER JOIN report_parameters
    ON orgunit.tenant = report_parameters.tenant
    AND (
            orgunit.id = report_parameters.root_orgunit
            OR orgunit. parent_ou  = report_parameters.root_orgunit
   )
INNER JOIN tenant
    ON orgunit.tenant = tenant.id;

-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
-- org-unit trials                                           -
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
CREATE TEMP TABLE orgunit_trials AS
SELECT
    trial.id AS trial,
    trial.publictitle AS trial_title,
    trialsite.orgunit,
    orgunit_relations.orgunit_name
FROM trial
INNER JOIN trialsite
    ON trial.id  = trialsite.trial
INNER JOIN orgunit_relations
    ON trialsite.orgunit = orgunit_relations.orgunit
    AND trial.tenant = (SELECT tenant FROM report_parameters)
;

-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
-- person to orgunit relations                               -
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
CREATE TEMPORARY TABLE person_orgunit_relations AS
SELECT
    orgunit.id AS orgunit,
    orgunit.parent_ou AS orgunit_parent,
    orgunit.name AS orgunit_name,
    person.id AS person,
    CONCAT(person.firstname , ' ', person.name) AS "name",
    person.email
FROM person
INNER JOIN person_ou
    ON person_ou.person = person.id
INNER JOIN orgunit_relations
    ON person_ou.orgunit = orgunit_relations.orgunit
INNER JOIN orgunit
    ON person_ou.orgunit = orgunit.id;


-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
-- org and trial roles                               -
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
CREATE TEMP TABLE org_trial_person_roles AS
SELECT DISTINCT site, trial, trial_title, person, orgtype, "role"
FROM (
    SELECT
        orgunit_relations.orgunit AS site,
        NULL AS trial,
        NULL AS trial_title,
        person_orgunit_relations.person,
        'site' AS orgtype,
        'clinic_user' AS "role"
    FROM person_orgunit_relations
    LEFT JOIN orgunit_relations
        ON person_orgunit_relations.orgunit = orgunit_relations.parent_orgunit
           OR orgunit_relations.parent_orgunit IS NULL 
    WHERE person_orgunit_relations.orgunit = (SELECT manager_orgunit FROM report_parameters)
    UNION
    SELECT
        person_orgunit_relations.orgunit AS site,
        trialsite_assistant.trial,
        trial.publictitle AS trial_title,
        trialsite_assistant.person,
        'trial' AS orgtype,
        'nurse_user' AS "role"
    FROM trialsite_assistant
    INNER JOIN person_orgunit_relations
        ON trialsite_assistant.person = person_orgunit_relations.person
    INNER JOIN trial
        ON trialsite_assistant.trial = trial.id
)
AS basic_trial_roles
ORDER BY basic_trial_roles.trial
;

-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
-- export                                                    -
-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
SELECT DISTINCT
    org_trial_person_roles.person,
    person_orgunit_relations.name,
    person_orgunit_relations.email,
    org_trial_person_roles.site,
    orgunit_relations.orgunit_name AS site_name,
    org_trial_person_roles.trial,
    org_trial_person_roles.trial_title,
    org_trial_person_roles.role    
FROM org_trial_person_roles
INNER JOIN person_orgunit_relations
    ON org_trial_person_roles.person = person_orgunit_relations.person
LEFT JOIN orgunit_relations
    ON org_trial_person_roles.site = orgunit_relations.orgunit
ORDER BY 
    org_trial_person_roles.person,
    org_trial_person_roles.site,
    org_trial_person_roles.trial
;
```



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

# Anmeldung
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

# Upgrade Notes
* Seit v3039: ENV "LICENCE_SECRET" Support eingestellt

* Seit v2794: ENV "LICENCE_SECRET" entfällt. Wird noch supported, sollte allerdings durch das neue Verschlüsselungsverfahren ersetzt werden. Hierfür muss die Datei "public.pem" über den Volume Abschnitt gesetzt werden

* Seit v2777: ENV `AI_SERVICE_FRONTEND_URL` und `AI_SERVICE_BACKEND_URL` wird durch `AI_BACKEND_BASE_URL` ersetzt. <br> 
  Backend Umgebungsvariable `AI_BACKEND_BEARER_TOKEN` hinzugefügt

* Seit v2342: ENV `OIDC_AUTH_NAME` entfernt