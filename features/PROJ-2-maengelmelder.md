# PROJ-2: Mängelmelder

**Status:** Planned
**Priorität:** P0 (MVP)
**Erstellt:** 2026-03-11

## Übersicht
Bürger können Mängel (Schlaglöcher, defekte Straßenlaternen, Vandalismus etc.) mit Foto und GPS-Standort melden. Die Verwaltung sieht alle Meldungen im Admin-Portal und kann den Status aktualisieren. Bürger sehen den Fortschritt ihrer Meldungen.

## User Stories

### US-1: Mangel melden
> Als Bürger möchte ich einen Mangel fotografieren und melden, damit die Gemeinde davon erfährt und ihn beheben kann.

**Akzeptanzkriterien:**
- [ ] Foto aufnehmen (Kamera) oder aus Galerie wählen (max. 3 Fotos)
- [ ] GPS-Koordinaten werden automatisch erfasst (Standortberechtigung erforderlich)
- [ ] Manuelle Standortkorrektur auf Karte möglich (Drag-Pin)
- [ ] Pflichtfelder: Kategorie, Kurzbeschreibung (max. 280 Zeichen)
- [ ] Optionales Feld: Detailbeschreibung
- [ ] Kategorien: Straße/Gehweg, Beleuchtung, Grünfläche, Vandalismus, Spielplatz, Sonstiges
- [ ] Meldung wird `municipality_id` des Bürgers zugeordnet
- [ ] Bestätigungs-Anzeige nach erfolgreicher Meldung
- [ ] Push-Benachrichtigung wenn Status sich ändert

### US-2: Eigene Meldungen verfolgen
> Als Bürger möchte ich den Status meiner Meldungen einsehen, damit ich weiß ob mein Anliegen bearbeitet wird.

**Akzeptanzkriterien:**
- [ ] Liste aller eigenen Meldungen (neueste zuerst)
- [ ] Statusanzeige mit Farb-Badge: Eingegangen (grau), In Bearbeitung (gelb), Erledigt (grün)
- [ ] Detail-Ansicht: Foto, Beschreibung, Karte, Statushistorie mit Zeitstempeln
- [ ] Öffentliche Meldungen anderer Bürger einsehbar (ohne Personendaten)

### US-3: Transparenz-Statistik
> Als Bürger möchte ich sehen wie schnell die Gemeinde Mängel behebt, damit ich Vertrauen in die Verwaltung aufbaue.

**Akzeptanzkriterien:**
- [ ] Öffentliches Dashboard: Gesamt-Meldungen, davon erledigt (%), Ø Bearbeitungszeit
- [ ] Aufschlüsselung nach Kategorie

### US-4: Verwaltung — Mängelbearbeitung (Admin-Portal)
> Als Verwaltungsmitarbeiter möchte ich eingehende Mängel einsehen und bearbeiten, damit ich sie zeitnah beheben kann.

**Akzeptanzkriterien:**
- [ ] Liste aller Meldungen der eigenen Gemeinde (gefiltert nach Status, Kategorie, Datum)
- [ ] Status ändern: Eingegangen → In Bearbeitung → Erledigt
- [ ] Interne Notiz hinzufügen (nur intern sichtbar)
- [ ] Öffentliche Antwort an Bürger senden (erscheint in Detail-Ansicht)
- [ ] Meldung als Duplikat markieren + Verweis auf Original
- [ ] CSV-Export der Meldungen

## Edge Cases
- GPS nicht verfügbar → Hinweis + manuelle Adresseingabe als Fallback
- Foto-Upload schlägt fehl → Meldung trotzdem ohne Foto einreichbar (mit Hinweis)
- Bürger meldet selben Mangel doppelt → System schlägt ähnliche Meldungen vor (Umkreis 50m)
- Bürger ist nicht eingeloggt → Weiterleitung zum Login
- Verwaltung ändert Status → Push-Benachrichtigung an meldenden Bürger

## Datenmodell

```sql
issue_reports (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  municipality_id uuid NOT NULL REFERENCES municipalities(id),
  citizen_id uuid NOT NULL REFERENCES auth.users(id),
  category text NOT NULL,         -- enum: road, lighting, greenspace, vandalism, playground, other
  title text NOT NULL,
  description text,
  public_response text,           -- Antwort der Verwaltung (für Bürger sichtbar)
  internal_note text,             -- nur intern sichtbar
  status text NOT NULL DEFAULT 'received',  -- received | in_progress | done | duplicate
  duplicate_of uuid REFERENCES issue_reports(id),
  latitude double precision NOT NULL,
  longitude double precision NOT NULL,
  address text,                   -- reverse-geocoded Adresse
  photo_urls text[],              -- Supabase Storage URLs
  resolved_at timestamptz,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  deleted_at timestamptz          -- Soft Delete
)

-- Status-Historie
issue_status_history (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  issue_id uuid NOT NULL REFERENCES issue_reports(id),
  old_status text,
  new_status text NOT NULL,
  changed_by uuid REFERENCES auth.users(id),
  note text,
  created_at timestamptz DEFAULT now()
)
```

## Screens (Mobile App)
1. `MaengelListe` — Karten-Ansicht + Liste-Toggle mit allen öffentlichen Meldungen
2. `MangelmeldungErstellen` — Foto + Karte + Formular (Stepper)
3. `MangelmeldungDetail` — Status-Timeline, Fotos, Karte, Verwaltungs-Antwort
4. `MeineMeldungen` — eigene Meldungen mit Status

## Admin-Portal Screens
1. `MaengelDashboard` — Übersicht + Statistik
2. `MaengelListe` — filterbare Tabelle
3. `MangelmeldungBearbeiten` — Status ändern, Notiz, Antwort

## Technische Hinweise
- Fotos: Supabase Storage, Bucket `issue-photos`, max. 5 MB pro Foto
- Karte: `react-native-maps` (iOS: Apple Maps, Android: Google Maps)
- GPS: `expo-location`
- Push-Benachrichtigungen: `expo-notifications`
- RLS: Bürger kann nur eigene Meldungen schreiben, alle `not deleted` lesen (public)
- Admin kann alle Meldungen der eigenen `municipality_id` lesen/schreiben

## Definition of Done
- [ ] Alle Akzeptanzkriterien erfüllt
- [ ] Foto-Upload in Supabase Storage funktioniert
- [ ] GPS-Erfassung auf iOS + Android getestet
- [ ] Push-Benachrichtigung bei Statusänderung funktioniert
- [ ] RLS-Policies korrekt (kein Cross-Tenant-Zugriff)
- [ ] Admin-Portal: Statusänderung + CSV-Export funktioniert

---

## Tech Design (Solution Architect)

> Vollständige Systemarchitektur: [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md)

### Wo lebt dieser Code?
```
apps/mobile/
  app/(tabs)/melden/
    index.tsx                    MaengelListe (Karte + Liste)
    erstellen.tsx                Stepper: Foto → Karte → Formular
    [id].tsx                     MangelmeldungDetail

apps/web/
  app/(dashboard)/issues/
    page.tsx                     MaengelListe Admin (Tabelle)
    [id]/page.tsx                Detail + Statusänderung

packages/shared/
  types/issue.ts                 IssueReport, IssueStatus (Mobile + Web)
  constants/issue-categories.ts  Kategorien (einmal definiert)

packages/api-client/
  issues.ts                      Erstellen, Lesen, Status-Update, CSV

supabase/functions/
  send-push-notification/        Ausgelöst bei Statusänderung in issue_reports
```

### Karten-Strategie
- iOS: Apple Maps (kostenlos, kein API-Key nötig)
- Android: Google Maps (kostenloser Kontingent für MVP ausreichend)
- Reverse Geocoding: Google Maps API oder Nominatim (OpenStreetMap, kostenlos)

### Foto-Upload-Flow
```
Nutzer wählt Foto
  → Komprimierung auf Gerät (max. 1 MB pro Foto, Qualität 80%)
  → Upload zu Supabase Storage (Bucket: issue-photos)
  → URL wird in issue_reports.photo_urls gespeichert
```

### Push-Benachrichtigung bei Statusänderung
- Supabase Edge Function reagiert auf DB-Trigger in `issue_status_history`
- Sendet Push via Expo Push API an den meldenden Bürger
- Falls Push-Token abgelaufen: Fehler wird geloggt, kein Absturz

### Abhängigkeiten (neue Pakete)
- `react-native-maps` — Kartenanzeige + Pin-Interaktion
- `expo-location` — GPS-Koordinaten
- `expo-camera` + `expo-image-picker` — Foto aufnehmen / auswählen
- `expo-notifications` — Push-Empfang
- `expo-image-manipulator` — Bild-Komprimierung vor Upload
