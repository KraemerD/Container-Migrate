# Container Migrate – Unraid Plugin

Ein-Klick-Migration von Docker-Containern zwischen mehreren Unraid-Servern im selben Netzwerk. Auf jedem Server installiert, kann jeder Container zu jedem verbundenen Peer verschoben werden – mit Vorschau und Bestätigung pro Schritt.

## Sicherheitsmodell

- **Der Quell-Server wird nie verändert oder gelöscht.** Container werden lokal nur gestoppt, Daten werden kopiert (nicht verschoben). Du schaltest den alten Server erst manuell ab, wenn der Test auf dem Ziel erfolgreich war.
- **Kein Schritt läuft ohne Bestätigung.** Die WebUI zeigt erst den vollständigen Plan inkl. Konflikt-Warnungen (Ports, fehlende Netzwerke, fehlendes Template). Jeden Schritt löst du einzeln aus.
- **Lokal & ohne Cloud.** Transport über SSH + rsync, beides auf Unraid vorhanden. Kein zusätzlicher Daemon, keine externen Dienste.

## Was migriert wird

| Teil | Quelle | Automatisch |
|------|--------|-------------|
| Template (XML) | `/boot/config/plugins/dockerMan/templates-user/my-*.xml` | ✓ |
| appdata | `/mnt/user/appdata/<container>` | ✓ |
| Weitere Bind-Volumes | aus `docker inspect` | ✓ |
| Ports / Netzwerk | Konflikt-Prüfung gegen Ziel | Warnung + manuelle Bestätigung |
| Container-Erstellung | aus Template auf dem Ziel | Template wird übertragen, Finalisierung im Docker-Tab |

## Architektur

```
WebUI (.page + migrate.js)
        │  AJAX
        ▼
api.php  ──►  Migrate.php   (Discovery, Plan-Bau, SSH-Tests – verändert nichts)
        │
        └──►  migrate.sh    (führt EINEN bestätigten Schritt aus: stop/rsync/create)
                  │ SSH + rsync
                  ▼
            Ziel-Unraid-Server (Peer)
```

## Installation (lokaler Test)

Auf dem Unraid-Server, ohne GitHub-Release:

```bash
# 1. Package und Plugin-Datei auf den Server kopieren (z.B. nach /boot/config/plugins/)
# 2. Package-Inhalt direkt entpacken zum Schnelltest:
installpkg /pfad/zu/container.migrate-2026.06.14.txz
chmod +x /usr/local/emhttp/plugins/container.migrate/scripts/migrate.sh
# 3. WebUI aufrufen: Settings → Utilities → Container Migrate
```

> Hinweis: Direkt entpackte Dateien gehen bei einem Reboot verloren (Unraid lädt das OS frisch). Für Persistenz das `.plg` über den Plugin-Manager installieren.

## Installation (echtes Plugin via .plg)

1. Repo auf GitHub anlegen (`KraemerD/container-migrate`), `.txz` als Release-Asset hochladen.
2. URLs im `.plg` zeigen bereits auf `KraemerD/container-migrate` – ggf. anpassen.
3. Auf jedem Unraid: **Plugins → Install Plugin** → Roh-URL der `container.migrate.plg` einfügen.

## SSH einrichten (einmalig pro Server-Paar)

1. In der WebUI auf **„SSH-Schlüssel anzeigen"** klicken (erzeugt einen ed25519-Key).
2. Den angezeigten Public Key auf **jedem Ziel-Server** in `/root/.ssh/authorized_keys` eintragen.
   Auf Unraid persistent machen über die `go`-Datei oder das „SSH"-Plugin, da `/root/.ssh` sonst beim Reboot zurückgesetzt wird.
3. In der Peer-Liste erscheint der Status dann als „verbunden".

## Bekannte Grenzen / Roadmap

- **Container-Erstellung** legt aktuell das Template bereit; die finale Erstellung bestätigst du im Docker-Tab des Ziels. Vollautomatisches Anlegen per Unraid-CLI ist der nächste Ausbau.
- **Custom Docker-Netzwerke** werden erkannt und gewarnt, aber nicht automatisch auf dem Ziel angelegt.
- **Datenbanken**: Der Stop-Schritt sorgt für konsistente Daten; bei sehr großen DBs lohnt ein Wartungsfenster.
- Geplant: paralleles Verschieben mehrerer Container, Rückweg-Migration (Rollback), Bandbreitenlimit für rsync.

## Build aus Quellcode

```bash
cd src
tar -cJf ../container.migrate-<version>.txz --owner=0 --group=0 usr install
```
