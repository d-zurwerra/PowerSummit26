# Projekt-Onboarding-Flow – Microsoft 365 & Power Platform

> Live-Build-Session vom Power Platform Summit (14. Juli 2026): Eine Power-Automate-Flow-Lösung, die das Onboarding neuer Projekte in einem Microsoft-365-Tenant automatisiert – inklusive Teams-Erstellung, Kanalstruktur, Planner-Plan, OneNote-Notizbuch und Genehmigungsprozess.

**Leitgedanke dieses Repos:** Wir zeigen echte Grenzen, keine polierten Workarounds. Alle dokumentierten Stolpersteine (fehlende Standard-Aktionen, Propagation-Delays, Auth-Einschränkungen) sind bewusst sichtbar gelassen – nicht wegoptimiert.

---

## Inhalt dieses Repos

- 📊 **Slides** der Session als einzelne Dateien (`/slides`)
- 📦 **Power Platform Solution** als Gesamtpaket – enthält Power App, alle Flows und Environment-Variablen (`/solution`)
- 📄 Diese Dokumentation

> Die Solution wird bewusst **als Ganzes** (nicht als einzelne Flow-/App-Exporte) abgelegt, da sie Environment-Variablen enthält, die beim Import zentral im Power Platform Admin Center konfiguriert werden müssen (z. B. Tenant-spezifische IDs, Resource-URIs für die Graph-API-Aufrufe).

### Import der Solution

1. Power Platform Admin Center → **Solutions** → **Import Solution**
2. Die `.zip`-Datei aus `/solution` auswählen
3. Beim Import die **Environment-Variablen** setzen (z. B. Site-/List-ID der SharePoint-Liste, Resource-URI für die HTTP-mit-Entra-ID-Verbindung)
4. Verbindungen (SharePoint, Teams, Planner, OneNote, HTTP mit Microsoft Entra ID) nach dem Import neu autorisieren

## Voraussetzungen

- Microsoft 365 Tenant mit Berechtigung zum Erstellen von Teams, Planner-Plänen, OneNote-Notizbüchern
- **Power Automate Premium-Lizenz** (wird für HTTP-Aktionen / Graph-API-Aufrufe benötigt, z. B. Planner-Plan-Erstellung)
- Microsoft Entra ID App-Registrierung für den Graph-API-Zugriff (App-Only-Auth via `ActiveDirectoryOAuth`)
- Berechtigung `Tasks.ReadWrite.All` (Application Permission) für die Planner-Plan-Erstellung über Graph

## ⚠️ Wichtig: SharePoint-Liste vor dem Import anlegen

Der Flow wird über eine **SharePoint-Liste** ausgelöst, die als Entkopplungsschicht dient (bewusst kein direkter Power-Apps-Trigger – so bleibt der Flow offen für spätere Integrationen mit externen Systemen). **Diese Liste muss vor dem Import der Solution manuell in SharePoint angelegt werden**, mit exakt folgendem Schema:

| Spaltenname | Typ | Hinweise |
|---|---|---|
| Projektname | Einzeilige Textzeile | |
| Teamtyp | Auswahl | Werte: `Intern`, `Extern` |
| Kundenname | Einzeilige Textzeile | |
| Projektleitung | Person oder Gruppe | Einzelauswahl |
| Projektteam | Person oder Gruppe | Mehrfachauswahl |
| Genehmigung erforderlich | Ja/Nein | |
| Status | Auswahl | Werte: `Neu`, `In Bearbeitung`, `Abgeschlossen`, `Fehler` |

> Nach dem Import der Solution muss der Flow auf diese Liste (Site- und List-ID) umgebogen werden, da SharePoint-Listen nicht automatisch mit der Solution mitgepackt werden.

## Bekannte Grenzen & bewusste Design-Entscheidungen

Diese Punkte werden in der Session/den Slides im Detail erklärt und sind absichtlich **nicht** wegabstrahiert:

- Es gibt keine Standard-Tier-Aktion "Create a plan" für Planner → Graph-API (HTTP mit Microsoft Entra ID) ist notwendig.
- Planner-Tab-Erstellung über Graph ist offiziell nicht unterstützt und fehleranfällig → im Flow als manueller Schritt markiert.
- OneNote-Notizbuch-Erstellung erfordert delegierte Authentifizierung; App-Only-Zugriff ist seit März 2025 von Microsoft blockiert → ebenfalls manueller Schritt.
- Die OneNote-Notizbuch-Benennung ist tenant-sprachabhängig und liegt in `SiteAssets`, nicht wie oft dokumentiert in `Shared Documents`/`General`.
- Propagation-Delays bei Teams-, Planner-Mitgliedschaft und dem Tabs-Endpunkt treten unabhängig voneinander auf → jeweils eigene Retry-Schleife (Do-Until) im Flow.
- Der "HTTP mit Microsoft Entra ID"-Connector zeigt in der UI nur ein Resource-URI-Feld (keine Client-ID/Secret-Felder) → die Entra-App-Registrierung wird nicht direkt darüber genutzt; stattdessen App-Only-Auth via `ActiveDirectoryOAuth` mit Credentials direkt im HTTP-Aktions-Body.

## Aufbau des Flows (Kurzüberblick)

1. SharePoint-Trigger (neuer Listeneintrag)
2. Variablen initialisieren, Switch auf `Teamtyp`
3. Bedingte Manager-Genehmigung (Scope + Terminate bei Ablehnung)
4. Status-Update
5. Teams-Erstellung mit Do-Until-Retry
6. Kanal-Umbenennung (Default → „01 - Allgemein") via Liste-Kanäle + Filter Array + Update-Kanal
7. Apply-to-each über restliche Kanäle (`skip(variables('Kanaele'), 1)`)
8. Planner-Plan via HTTP/Graph, Bucket-Schleife
9. OneNote-Abschnitts-Schleife
10. Mitglieder-Hinzufügen, Willkommens-Post

## Lizenz

Dieses Projekt steht unter der [MIT-Lizenz](./LICENSE).

## Feedback & Kontakt

Fragen, Anmerkungen oder eigene Erfahrungen mit denselben Stolpersteinen? Gerne über Issues in diesem Repo oder [dein bevorzugter Kontaktweg, z. B. LinkedIn/X].
