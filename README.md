# CiviCRM-Veriff-Identity-Information-added-to-member
This is a CiviCRM extension to add identity information to a new member.

---

# Veriff Media für Kontakte (CiviCRM-Extension) – v2.2

Ruft **Selfie**, **Ausweis-Vorderseite** und (falls vorhanden) **-Rückseite**
aus einer Veriff-Session ab und legt sie als **File-Custom-Fields** am
CiviCRM-Kontakt ab.

## ⚠️ Vor der Installation lesen

**Bitte zuerst auf einem TEST-/Staging-System installieren, nicht direkt
produktiv.** Eine frühere Version dieser Extension hat auf einem Live-System
(CiviCRM 6.17 / Smarty5) einen fatalen Fehler ausgelöst. Diese Version behebt
die bekannte Ursache und ist syntaktisch mit PHP 8.4 geprüft – ein echter
Testlauf gegen deine konkrete CiviCRM-Version bleibt aber unverzichtbar.

**Immer über die CiviCRM-Oberfläche deinstallieren** (Extensions → Uninstall),
niemals nur den Ordner löschen – sonst bleiben „Geist"-Einträge in der
Datenbank zurück.

## Was in v2.2 korrigiert wurde

- **Automatischer Import neu umgesetzt.** Der frühere Scheduled Job scheiterte
  in CiviCRM 6.17 mit „API (Job, veriffmedia_sync) does not exist" – die
  Auflösung selbstgebauter APIv3-Job-Actions ist dort unzuverlässig. Der
  automatische Import läuft jetzt über `hook_civicrm_cron` und wird über
  Einstellungen gesteuert (an/aus, Intervall, Limit). Robuster und ohne
  API-Auflösungsproblem.

### Automatischen Import aktivieren

Der automatische Import ist standardmäßig **AUS**. Zum Einschalten die
Einstellung `veriffmedia_auto_enabled` auf 1 setzen – am einfachsten über den
API4 Explorer (`civicrm/api4`), Entity **Setting**, Action **set**:

```
civicrm/api4/#/explorer/Setting/set
```
Dort als Wert eintragen: `veriffmedia_auto_enabled = true`. Optional:
`veriffmedia_auto_interval_min` (Minuten zwischen Läufen, Standard 60) und
`veriffmedia_auto_limit` (max. Kontakte pro Lauf, Standard 100).

Danach läuft der Import automatisch bei jedem CiviCRM-Cron-Tick, sofern das
Intervall verstrichen ist. **Voraussetzung: ein laufender CiviCRM-Cron.**

## Was in v2.1 korrigiert wurde

- **File-Felder werden jetzt korrekt befüllt.** Der Bildabruf lief, aber das
  Ablegen im Feld scheiterte mit „Array hat nicht den richtigen Feld Daten Typ:
  File". Ursache war die alte Array-Übergabe an `CustomValueTable::setValues()`.
  Jetzt wird ein echter `civicrm_file`-Datensatz angelegt, per
  `civicrm_entity_file` mit dem Kontakt verknüpft und die File-ID als Feldwert
  gesetzt.
- **Einstellungsseite entfernt.** Sie war nur Kosmetik (Zugangsdaten kommen
  ohnehin aus dem WordPress-Plugin) und verursachte nach dem Smarty5-Fix einen
  Template-Ladefehler. Die Zugangsdaten-Prüfung entfällt damit; die Extension
  liest die Daten weiterhin automatisch.

## Was in v2.0 korrigiert wurde

- **Smarty5-Ladefehler behoben (Kernfix):** Die civix-Boilerplate manipulierte
  Smartys `template_dir`. Unter Smarty5 (CiviCRM 6.x) ist diese Eigenschaft
  geschützt; der Zugriff zerstörte die Template-Verzeichnisliste, sodass
  CiviCRM nicht einmal mehr seine eigenen Kern-Templates laden konnte. Diese
  Manipulation wurde ersatzlos entfernt – Core registriert Extension-Templates
  ohnehin automatisch.
- **Keine eigene APIv4-Entity mehr:** Die frühere `VeriffMedia`-APIv4-Entity
  war unnötig (der manuelle Button und der Cron-Job nutzen den Importer direkt)
  und hatte bei unsauberer Deinstallation einen „Geist"-Eintrag hinterlassen.
  Entfernt – weniger Angriffsfläche.
- **Scheduled Job standardmäßig DEAKTIVIERT:** Der Job importiert nicht sofort
  nach der Installation. Du aktivierst ihn bewusst, nachdem der manuelle Import
  am Kontakt getestet ist.
- **Überflüssige `search_kit`-Abhängigkeit entfernt** (hatte zudem eine falsche
  Versionsangabe, die die Installation blockieren konnte).
- **Defensive Härtung:** Der Datenbankzugriff auf die WordPress-Tabelle prüft
  jetzt deren Existenz und ist so gekapselt, dass er die Kontakt-Detailseite
  nie blockieren kann.

## Voraussetzungen

1. CiviCRM auf **derselben** WordPress-Installation wie das Plugin „CiviCRM
   Veriff Membership Verification" (die Extension liest dessen `$wpdb`-Tabelle
   und `get_option()`).
2. Dieses Plugin ist installiert und mit gültigen Veriff-Zugangsdaten
   konfiguriert.
3. Für den automatischen Import: ein laufender CiviCRM-Cron. Der manuelle
   Button funktioniert auch ohne Cron.

## Installation

1. ZIP entpacken, Ordner `de.gutmenschen.veriffmedia` nach
   `wp-content/uploads/civicrm/ext/` kopieren.
2. In CiviCRM unter **Administer → System Settings → Extensions** installieren.
   Dabei werden automatisch angelegt:
   - drei File-Custom-Fields am Kontakt (Gruppe „Veriff-Verifikation"):
     Selfie, Ausweis Vorderseite, Ausweis Rückseite;
   - ein **deaktivierter** Scheduled Job „Veriff: Ausweisbilder importieren".
3. Zugangsdaten: in der Regel nichts zu tun – werden aus dem WordPress-Plugin
   (`civiveriff_settings`) gelesen. Bestätigung unter *Datenverwaltung und
   Masken anpassen → Veriff Media – Einstellungen*.

## Nutzung

**Manuell (empfohlen zum Start):** Kontakt öffnen → Panel „Veriff-Verifikation"
→ *Ausweisbilder abrufen*. Die Session wird automatisch zum Kontakt ermittelt.

**Automatisch:** Wenn der manuelle Weg funktioniert, den Scheduled Job unter
*Administer → System Settings → Scheduled Jobs* **aktivieren** (Enable) und mit
*Execute now* testen. Er importiert dann regelmäßig fehlende Bilder für
verifizierte Kontakte.

**Per API (optional):**
```php
// Ganzen Sync-Lauf auslösen (wie der Cron-Job):
civicrm_api3('Job', 'Veriffmedia_sync', ['limit' => 100]);
```
(Einzelne Kontakte lassen sich programmatisch über die Klasse
`CRM_Veriffmedia_Importer` importieren.)

## Zwei oder drei Bilder – beides ist gültig

Nicht jeder Kontakt hat drei Bilder; manche haben nur Selfie + Vorderseite.
Importiert wird immer nur, was Veriff für die Session tatsächlich liefert.
Fehlt die Rückseite, bleibt das Feld leer – kein Fehler. Der Cron-Job wertet
einen Kontakt als vollständig, sobald alle vorhandenen Bilder da sind, und
arbeitet ihn nicht bei jedem Lauf erneut ab.

## Datenschutz (DSGVO)

Ausweiskopien und Selfies sind besonders schützenswerte Daten. Sichere das
CiviCRM-`custom`-Upload-Verzeichnis gegen öffentlichen Zugriff, beschränke die
Sichtbarkeit der Feldgruppe auf berechtigte Rollen, und überlege, wie lange die
Dokumente aufbewahrt werden müssen (Veriff hält die Originale ohnehin vor).
Bei Deinstallation werden die Custom-Felder bewusst NICHT automatisch gelöscht,
um Datenverlust zu vermeiden; der Scheduled Job wird entfernt.

## Wichtiger Hinweis zum Mailchimp-Konflikt

Unabhängig von dieser Extension kann bei jeder Extension-Installation ein
Rebuild ausgelöst werden. Falls das Modul `mailchimpsync` aktiv ist, kann dabei
der bekannte Konflikt `activity_type: Value already exists: 52` auftreten. Ist
`mailchimpsync` deaktiviert, tritt er nicht auf.
