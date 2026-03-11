# PROJ-5: Gemeinde Admin-Portal — Grundfunktionen

**Status:** Planned
**Priorität:** P0 (MVP)
**Erstellt:** 2026-03-11

## Übersicht
Das webbasierte Admin-Portal ermöglicht Gemeinde-Mitarbeitern die Verwaltung aller Inhalte und Services für ihre Gemeinde. Das Portal ist mandantengetrennt — jede Gemeinde sieht nur ihre eigenen Daten. Der System-Admin kann neue Gemeinden anlegen und Feature-Flags aktivieren.

## User Stories

### US-1: Gemeinde-Admin Login
> Als Gemeinde-Admin möchte ich mich sicher einloggen, damit nur autorisierte Mitarbeiter Zugriff haben.

**Akzeptanzkriterien:**
- [ ] Login via E-Mail + Passwort (kein OAuth im Admin-Portal)
- [ ] Passwort zurücksetzen via E-Mail
- [ ] Session-Timeout nach 8h Inaktivität
- [ ] Fehlgeschlagene Login-Versuche: max. 5, dann 15 Minuten Sperrung (Rate Limiting)
- [ ] Audit-Log: Login/Logout wird protokolliert

### US-2: Dashboard
> Als Admin möchte ich eine Übersicht über die wichtigsten Kennzahlen sehen, damit ich den Status der Gemeinde-App kenne.

**Akzeptanzkriterien:**
- [ ] Aktive Nutzer (gesamt + letzten 30 Tage)
- [ ] Offene Mängelmeldungen (Anzahl + Ø Bearbeitungszeit)
- [ ] Offene Chat-Anfragen (Anzahl + Ø Antwortzeit)
- [ ] Nächste 5 Events
- [ ] Letzte 5 News-Beiträge
- [ ] Nur Daten der eigenen Gemeinde

### US-3: Nutzerverwaltung & Rollen
> Als Admin möchte ich Rollen vergeben und Mitarbeiter einladen, damit nur befugte Personen bestimmte Funktionen nutzen können.

**Akzeptanzkriterien:**
- [ ] Mitarbeiter per E-Mail einladen (Supabase Auth Magic Link)
- [ ] Rollen: Admin, Redakteur, Verwaltung/Stadtrat
- [ ] Rolle ändern, Mitarbeiter deaktivieren
- [ ] Übersicht aller aktiven Mitarbeiter + Rollen + letzter Login
- [ ] Admin kann sich selbst nicht die Admin-Rolle entziehen
- [ ] Audit-Log für Rollenänderungen

### US-4: White-Label / Branding konfigurieren
> Als Admin möchte ich Logo, Farben und Namen meiner Gemeinde konfigurieren, damit die App das Corporate Design der Gemeinde trägt.

**Akzeptanzkriterien:**
- [ ] Logo hochladen (PNG/SVG, max. 2 MB)
- [ ] Primärfarbe + Sekundärfarbe (HEX-Picker)
- [ ] Gemeinde-Anzeigename
- [ ] Vorschau der Änderungen vor dem Speichern
- [ ] Änderungen sofort in der Bürger-App sichtbar

### US-5: Feature-Flags
> Als Admin möchte ich einzelne Module aktivieren/deaktivieren, damit nur bezahlte/gewünschte Features aktiv sind.

**Akzeptanzkriterien:**
- [ ] Toggle-Liste aller verfügbaren Module (News, Events, Mängelmelder, Chat, Termine, etc.)
- [ ] Änderungen sofort wirksam (kein App-Restart nötig)
- [ ] Deaktivierte Module sind in der Bürger-App nicht sichtbar
- [ ] Audit-Log für Feature-Flag-Änderungen

### US-6: Audit-Log
> Als Admin möchte ich alle wichtigen Aktionen protokolliert sehen, damit ich Änderungen nachvollziehen kann.

**Akzeptanzkriterien:**
- [ ] Log-Einträge: Zeitstempel, Nutzer, Aktion, betroffene Entität
- [ ] Filterbar nach Nutzer, Aktion, Zeitraum
- [ ] Audit-Log ist read-only (nicht änderbar)
- [ ] Export als CSV

## Edge Cases
- Admin löscht sich selbst → Fehlermeldung, mindestens ein Admin muss existieren
- Feature-Flag deaktiviert während Nutzer das Modul gerade nutzt → graceful degradation in App
- Logo zu groß → Fehlermeldung mit Größenlimit
- Mitarbeiter-Einladungs-Link abgelaufen (48h) → erneut einladen möglich

## Datenmodell

```sql
municipality_staff (
  id uuid PRIMARY KEY REFERENCES auth.users(id),
  municipality_id uuid NOT NULL REFERENCES municipalities(id),
  role text NOT NULL,             -- admin | editor | staff
  invited_by uuid REFERENCES auth.users(id),
  is_active boolean DEFAULT true,
  last_login_at timestamptz,
  created_at timestamptz DEFAULT now()
)

municipality_settings (
  municipality_id uuid PRIMARY KEY REFERENCES municipalities(id),
  display_name text NOT NULL,
  logo_url text,
  primary_color text DEFAULT '#3B82F6',
  secondary_color text DEFAULT '#1E40AF',
  updated_at timestamptz DEFAULT now()
)

feature_flags (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  municipality_id uuid NOT NULL REFERENCES municipalities(id),
  feature_key text NOT NULL,      -- news | events | issues | chat | appointments | waste | etc.
  is_enabled boolean DEFAULT true,
  updated_at timestamptz DEFAULT now(),
  UNIQUE(municipality_id, feature_key)
)

audit_logs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  municipality_id uuid NOT NULL REFERENCES municipalities(id),
  user_id uuid REFERENCES auth.users(id),
  action text NOT NULL,           -- login | logout | create | update | delete | role_change | etc.
  entity_type text,               -- news_post | event | issue | chat_thread | etc.
  entity_id uuid,
  old_value jsonb,
  new_value jsonb,
  ip_address text,
  created_at timestamptz DEFAULT now()
)
```

## Admin-Portal Screens
1. `Login` — E-Mail + Passwort
2. `Dashboard` — KPI-Kacheln + Quick-Links
3. `Nutzerverwaltung` — Mitarbeiter-Tabelle + Einladen
4. `Branding` — Logo + Farben + Vorschau
5. `FeatureFlags` — Toggle-Liste der Module
6. `AuditLog` — filterbare Log-Tabelle

## Technische Hinweise
- Next.js App Router mit Server Components
- shadcn/ui für alle UI-Komponenten
- Middleware für Auth-Check auf allen `/admin/*` Routes
- Supabase RLS: Staff sieht nur eigene `municipality_id`
- Rate Limiting: Supabase Edge Function oder Middleware
- Farbvorschau: CSS Custom Properties live aktualisieren

## Definition of Done
- [ ] Login + Session-Timeout funktioniert
- [ ] Rate Limiting bei Login-Versuchen aktiv
- [ ] Alle CRUD-Operationen via RLS auf eigene Gemeinde beschränkt
- [ ] Feature-Flags sofort in Bürger-App wirksam
- [ ] Audit-Log wird für alle kritischen Aktionen befüllt
- [ ] Branding-Änderungen in Echtzeit sichtbar
