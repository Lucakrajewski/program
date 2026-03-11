# PROJ-3: News & Events

**Status:** Planned
**Priorität:** P0 (MVP)
**Erstellt:** 2026-03-11

## Übersicht
Gemeinden veröffentlichen News-Beiträge und Veranstaltungen. Bürger erhalten Push-Erinnerungen und können Events als "Interessiert" markieren. Vereine können eigene Events anlegen (optional, via Admin-Freigabe).

## User Stories

### US-1: News lesen
> Als Bürger möchte ich aktuelle Neuigkeiten meiner Gemeinde lesen, damit ich informiert bleibe.

**Akzeptanzkriterien:**
- [ ] News-Feed mit neuesten Beiträgen (neueste zuerst, paginiert)
- [ ] Beitrags-Detail: Titel, Bild, Text (Markdown), Datum, Kategorie
- [ ] Kategorien: Allgemein, Baustellen, Verkehr, Kultur, Sonstiges
- [ ] Suche nach Stichwort
- [ ] Teilen-Funktion (nativer Share-Dialog)
- [ ] Nur Beiträge der eigenen Gemeinde sichtbar

### US-2: Veranstaltungskalender
> Als Bürger möchte ich kommende Veranstaltungen sehen und mich erinnern lassen, damit ich nichts verpasse.

**Akzeptanzkriterien:**
- [ ] Kalender-Ansicht (Monats- + Listenansicht)
- [ ] Event-Detail: Titel, Beschreibung, Datum/Uhrzeit, Ort, Bild, Organisator
- [ ] "Interessiert"-Button → Event wird in "Meine Events" gespeichert
- [ ] Push-Erinnerung: 1 Tag vorher (opt-in)
- [ ] Vergangene Events nicht mehr prominent angezeigt
- [ ] Verein als Organisator anzeigbar

### US-3: Ticketreservierung (MVP-light)
> Als Bürger möchte ich für ein Event meine Teilnahme bestätigen, damit der Veranstalter die Teilnehmerzahl kennt.

**Akzeptanzkriterien:**
- [ ] Einfache Anmeldung (Name + E-Mail vorausgefüllt aus Profil)
- [ ] Max. Teilnehmerzahl pro Event konfigurierbar
- [ ] Warteliste wenn ausgebucht
- [ ] Bestätigungs-E-Mail via Supabase Edge Function
- [ ] Stornierung bis 24h vor Event möglich
- [ ] Kein Bezahlsystem (MVP — Non-Goal)

### US-4: Redakteur — Inhalte verwalten (Admin-Portal)
> Als Redakteur möchte ich News und Events erstellen und verwalten, damit Bürger informiert sind.

**Akzeptanzkriterien:**
- [ ] CRUD für News-Beiträge mit Rich-Text-Editor (TipTap oder Markdown)
- [ ] CRUD für Events: Titel, Beschreibung, Datum, Uhrzeit, Ort, Bild, Max-Teilnehmer
- [ ] Beiträge können als Entwurf gespeichert oder sofort veröffentlicht werden
- [ ] Beiträge können auf Datum terminiert werden (scheduled publish)
- [ ] Bild-Upload via Supabase Storage
- [ ] Teilnehmerliste eines Events einsehen + CSV-Export

## Edge Cases
- Event in der Vergangenheit: automatisch als "abgelaufen" markiert
- Max-Teilnehmer erreicht: Warteliste-Eintrag statt Anmeldung
- Push-Benachrichtigung wenn App deinstalliert: fehlschlägt gracefully
- Bild-Upload > 10 MB: Fehlermeldung mit Größenlimit-Hinweis

## Datenmodell

```sql
news_posts (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  municipality_id uuid NOT NULL REFERENCES municipalities(id),
  author_id uuid REFERENCES auth.users(id),
  title text NOT NULL,
  content text NOT NULL,          -- Markdown
  category text DEFAULT 'general',
  image_url text,
  is_published boolean DEFAULT false,
  published_at timestamptz,       -- für scheduled publish
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz,
  deleted_at timestamptz
)

events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  municipality_id uuid NOT NULL REFERENCES municipalities(id),
  author_id uuid REFERENCES auth.users(id),
  organizer_name text,            -- z.B. Verein
  title text NOT NULL,
  description text,
  image_url text,
  location text,
  starts_at timestamptz NOT NULL,
  ends_at timestamptz,
  max_attendees integer,
  is_published boolean DEFAULT false,
  ticket_required boolean DEFAULT false,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz,
  deleted_at timestamptz
)

event_registrations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id uuid NOT NULL REFERENCES events(id),
  citizen_id uuid NOT NULL REFERENCES auth.users(id),
  is_waitlist boolean DEFAULT false,
  registered_at timestamptz DEFAULT now(),
  cancelled_at timestamptz,
  UNIQUE(event_id, citizen_id)
)

event_reminders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id uuid NOT NULL REFERENCES events(id),
  citizen_id uuid NOT NULL REFERENCES auth.users(id),
  remind_at timestamptz NOT NULL,
  sent_at timestamptz,
  UNIQUE(event_id, citizen_id)
)
```

## Screens (Mobile App)
1. `NewsListe` — scrollbarer Feed mit Bild-Kacheln
2. `NewsDetail` — Vollansicht Beitrag
3. `EventKalender` — Monats- + Listenansicht
4. `EventDetail` — Infos + Anmeldung + Erinnerung
5. `MeineEvents` — Angemeldete Events

## Admin-Portal Screens
1. `NewsUebersicht` — Tabelle mit Status + Quick-Actions
2. `NewsEditor` — Rich-Text-Editor mit Vorschau
3. `EventUebersicht` — Tabelle mit Teilnehmerzahl
4. `EventEditor` — Formular + Bild-Upload
5. `EventTeilnehmer` — Teilnehmerliste + CSV-Export

## Technische Hinweise
- Rich-Text-Editor Admin: TipTap (React)
- Push-Benachrichtigungen: Supabase Edge Function + Expo Push Notifications API
- Bild-Upload: Supabase Storage, Bucket `content-images`
- Paginierung: Cursor-based (after/before UUID)
- RLS: Nur veröffentlichte Posts eigener Gemeinde für Bürger lesbar

## Definition of Done
- [ ] Alle Akzeptanzkriterien erfüllt
- [ ] Push-Erinnerung 1 Tag vor Event getestet
- [ ] Teilnehmerliste + CSV-Export funktioniert
- [ ] Scheduled publish funktioniert
- [ ] RLS-Policies korrekt

---

## Tech Design (Solution Architect)

> Vollständige Systemarchitektur: [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md)

### Wo lebt dieser Code?
```
apps/mobile/
  app/(tabs)/home/
    index.tsx                    News-Feed (Start-Tab)
    [id].tsx                     News-Beitrag Detail
  app/(tabs)/events/
    index.tsx                    Kalender + Listenansicht
    [id].tsx                     Event-Detail + Anmeldung

apps/web/
  app/(dashboard)/news/
    page.tsx                     News-Übersicht (Tabelle)
    new/page.tsx                 Rich-Text-Editor
    [id]/edit/page.tsx           Bearbeiten
  app/(dashboard)/events/
    page.tsx                     Event-Übersicht
    [id]/attendees/page.tsx      Teilnehmerliste

packages/shared/
  types/news.ts                  NewsPost-Typ
  types/event.ts                 Event + EventRegistration-Typ

supabase/functions/
  send-reminder/                 Scheduler: täglich prüfen, Push 24h vor Event
  send-confirmation-email/       Ausgelöst bei event_registrations INSERT
```

### Scheduled-Publish-Mechanismus
```
Redakteur setzt published_at = "2026-04-01 09:00"
→ Supabase Edge Function läuft stündlich (Cron)
→ Prüft: published_at <= jetzt AND is_published = false
→ Setzt is_published = true
→ App zeigt Beitrag sofort
```

### Kalender-Darstellung (Mobile)
- Listenansicht: Standard (einfachste Implementierung für MVP)
- Monatsansicht: `react-native-calendars` (leichtgewichtig)
- Filter: Kategorie-Chips über der Liste

### Abhängigkeiten (neue Pakete)
- `@tiptap/react` — Rich-Text-Editor (Admin-Portal, bereits evaluiert)
- `react-native-calendars` — Kalender-Komponente (Mobile)
- `date-fns` — Datumsformatierung mit deutscher Locale
