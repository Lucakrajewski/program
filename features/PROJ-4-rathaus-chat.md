# PROJ-4: Rathaus-Chat

**Status:** Planned
**Priorität:** P0 (MVP)
**Erstellt:** 2026-03-11

## Übersicht
Bürger können direkt mit der Gemeindeverwaltung kommunizieren. Der Chat ist kategorie-basiert (Bürgerbüro, Bauamt, etc.), hat ein Statussystem und benachrichtigt beide Seiten per Push bei neuen Nachrichten. Interne Notizen sind nur für Verwaltungsmitarbeiter sichtbar.

## User Stories

### US-1: Chat-Anfrage stellen
> Als Bürger möchte ich eine Anfrage an die Gemeinde stellen, damit ich schnell Antworten erhalte ohne persönlich erscheinen zu müssen.

**Akzeptanzkriterien:**
- [ ] Neuen Chat-Thread erstellen: Kategorie + Betreff + Nachricht
- [ ] Kategorien: Bürgerbüro, Bauamt, Ordnungsamt, Soziales, Allgemein
- [ ] Anhänge möglich: Fotos + PDFs (max. 10 MB gesamt)
- [ ] Thread-Status: Offen, In Bearbeitung, Geschlossen
- [ ] Bürger sieht alle eigenen Threads mit Status
- [ ] Push-Benachrichtigung bei neuer Antwort
- [ ] Geschlossene Threads können nicht mehr beantwortet werden
- [ ] Thread kann erneut geöffnet werden (Bürger sendet neue Nachricht → Status → Offen)

### US-2: Chat-Verlauf einsehen
> Als Bürger möchte ich den Verlauf meiner Anfragen einsehen, damit ich den Kontext behalte.

**Akzeptanzkriterien:**
- [ ] Alle eigenen Threads in einer Liste (neueste zuerst)
- [ ] Thread-Detail: Alle Nachrichten chronologisch, Statusänderungen sichtbar
- [ ] Nachrichten zeigen Zeitstempel + Absender (Bürger oder "Gemeindeverwaltung")
- [ ] Ungelesene Nachrichten werden als Badge angezeigt

### US-3: Verwaltung — Chat bearbeiten (Admin-Portal)
> Als Verwaltungsmitarbeiter möchte ich eingehende Anfragen beantworten und intern notieren, damit ich effizient arbeiten kann.

**Akzeptanzkriterien:**
- [ ] Alle offenen Threads der eigenen Gemeinde sehen (gefiltert nach Kategorie, Status)
- [ ] Thread beantworten (Antwort für Bürger sichtbar)
- [ ] Interne Notiz hinzufügen (nur Admin-Portal sichtbar, nicht für Bürger)
- [ ] Thread-Status ändern: Offen → In Bearbeitung → Geschlossen
- [ ] Thread einer anderen Abteilung/Kategorie zuweisen
- [ ] Durchschnittliche Antwortzeit im Dashboard sichtbar
- [ ] Ungelesene Threads als Badge im Sidebar

### US-4: Simpler Antwort-Bot (MVP-Basis)
> Als Verwaltung möchte ich häufige Fragen automatisch beantworten lassen, damit Mitarbeiter entlastet werden.

**Akzeptanzkriterien:**
- [ ] Regelbasierter Bot (kein KI): Stichwort-Matching auf Betreff
- [ ] Bot-Antworten konfigurierbar im Admin-Portal (FAQ-Verwaltung)
- [ ] Bot antwortet sofort mit Standardantwort + Hinweis "Mitarbeiter prüft ebenfalls"
- [ ] Bot-Antworten sind als "Automatische Antwort" gekennzeichnet
- [ ] Bot kann pro Kategorie deaktiviert werden

## Edge Cases
- Bürger sendet Nachricht an geschlossenen Thread → Fehlermeldung + Option Thread neu öffnen
- Anhang zu groß → Fehlermeldung mit Limit-Hinweis
- Verwaltungsmitarbeiter antwortet an falsche Kategorie → Weiterleitungs-Funktion
- Push-Token abgelaufen → Notification schlägt gracefully fehl, kein Fehler für Nutzer

## Datenmodell

```sql
chat_threads (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  municipality_id uuid NOT NULL REFERENCES municipalities(id),
  citizen_id uuid NOT NULL REFERENCES auth.users(id),
  category text NOT NULL,         -- town_hall | building | public_order | social | general
  subject text NOT NULL,
  status text NOT NULL DEFAULT 'open',  -- open | in_progress | closed
  assigned_to uuid REFERENCES auth.users(id),  -- Verwaltungs-Mitarbeiter
  last_message_at timestamptz,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz,
  deleted_at timestamptz
)

chat_messages (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  thread_id uuid NOT NULL REFERENCES chat_threads(id) ON DELETE CASCADE,
  sender_id uuid REFERENCES auth.users(id),   -- NULL = Bot
  sender_type text NOT NULL,                   -- citizen | staff | bot
  content text NOT NULL,
  is_internal boolean DEFAULT false,           -- interne Notiz
  attachment_urls text[],
  read_at timestamptz,
  created_at timestamptz DEFAULT now()
)

chat_bot_rules (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  municipality_id uuid NOT NULL REFERENCES municipalities(id),
  category text,                  -- NULL = alle Kategorien
  keywords text[] NOT NULL,       -- Stichwörter für Matching
  response text NOT NULL,
  is_active boolean DEFAULT true,
  created_at timestamptz DEFAULT now()
)
```

## Screens (Mobile App)
1. `ChatListe` — alle eigenen Threads mit Status + Ungelesen-Badge
2. `ChatThread` — Nachrichten-Verlauf + Eingabe
3. `NeuerThread` — Kategorie + Betreff + Nachricht + Anhang

## Admin-Portal Screens
1. `ChatDashboard` — Offene Anfragen + Ø Antwortzeit + Kategorie-Breakdown
2. `ChatListe` — Filterbare Tabelle aller Threads
3. `ChatThread` — Antwort + interne Notiz + Statusänderung
4. `BotRegelnVerwalten` — FAQ-Einträge CRUD

## Technische Hinweise
- Echtzeit-Updates: Supabase Realtime (Postgres LISTEN/NOTIFY)
- Anhänge: Supabase Storage, Bucket `chat-attachments`
- Push: Expo Push Notifications bei neuer Nachricht
- Bot-Matching: serverseitig in Supabase Edge Function (bei Thread-Erstellung)
- RLS: Bürger liest/schreibt nur eigene Threads; Staff liest alle Threads eigener `municipality_id`

## Definition of Done
- [ ] Alle Akzeptanzkriterien erfüllt
- [ ] Echtzeit-Updates (Supabase Realtime) funktionieren
- [ ] Interne Notizen nur für Staff sichtbar (RLS verifiziert)
- [ ] Bot-Regelwerk im Admin-Portal konfigurierbar
- [ ] Push-Benachrichtigungen bei neuer Nachricht getestet
- [ ] Anhang-Upload und -Download funktionieren
