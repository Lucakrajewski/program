# PROJ-1: Onboarding & Gemeinde-Auswahl

**Status:** Planned
**Priorität:** P0 (MVP)
**Erstellt:** 2026-03-11

## Übersicht
Beim ersten App-Start durchläuft der Nutzer einen Pflicht-Onboarding-Flow: Gemeinde auswählen → Registrieren → Bürgerprofil anlegen. Ohne abgeschlossenes Onboarding ist kein Zugriff auf die App möglich.

## User Stories

### US-1: Gemeinde auswählen
> Als neuer Nutzer möchte ich meine Gemeinde auswählen, damit alle Inhalte der App für meine Gemeinde angezeigt werden.

**Akzeptanzkriterien:**
- [ ] Beim ersten App-Start wird die Gemeinde-Auswahl angezeigt (kein Überspringen möglich)
- [ ] Suche nach Name oder PLZ möglich
- [ ] Nur aktive Gemeinden (is_active = true) werden angezeigt
- [ ] Auswahl wird lokal gespeichert (AsyncStorage) und in der DB verknüpft
- [ ] Gemeinde kann später in den Einstellungen geändert werden
- [ ] Die gewählte `municipality_id` wird für alle folgenden API-Calls mitgesendet

### US-2: Registrierung
> Als neuer Nutzer möchte ich mich registrieren, damit ich einen persönlichen Account erhalte.

**Akzeptanzkriterien:**
- [ ] Registrierung via E-Mail + Passwort
- [ ] Registrierung via Google OAuth
- [ ] Registrierung via Apple Sign-In (iOS Pflicht für App Store)
- [ ] E-Mail-Verifikation wird versendet (Supabase Auth)
- [ ] Bei bereits existierender E-Mail: klare Fehlermeldung + "Login"-Link
- [ ] Passwort-Anforderungen: min. 8 Zeichen
- [ ] DSGVO-Hinweis + Datenschutzerklärung akzeptieren (Pflicht-Checkbox)

### US-3: Bürgerprofil anlegen
> Als registrierter Nutzer möchte ich mein Bürgerprofil anlegen, damit ich Verwaltungsservices nutzen kann.

**Akzeptanzkriterien:**
- [ ] Pflichtfelder: Vorname, Nachname, Geburtsdatum, Straße, Hausnummer, PLZ, Ort
- [ ] Optionales Feld: Telefonnummer
- [ ] Geburtsdatum: Datepicker, min. 14 Jahre (DSGVO)
- [ ] PLZ wird gegen bekannte PLZ validiert (oder freies Textfeld)
- [ ] Profil ist nach Anlage bearbeitbar in "Mein Profil"
- [ ] Profildaten werden verschlüsselt in `citizen_profiles` gespeichert
- [ ] `municipality_id` wird automatisch aus der Gemeinde-Auswahl übernommen

## Edge Cases
- Nutzer bricht Onboarding ab → App bleibt auf Onboarding-Screen
- Netzwerkfehler bei Registrierung → Fehlermeldung + Retry-Option
- Apple Sign-In gibt keine E-Mail zurück → Fallback auf E-Mail-Eingabe
- Gemeinde wird während Onboarding deaktiviert → Fehlermeldung, andere Gemeinde wählen
- Nutzer wechselt Gemeinde → alle gemeindegebundenen Daten (Müllkalender etc.) werden neu geladen

## Datenmodell

```sql
-- municipalities (bereits vorhanden, read-only für Bürger)
municipalities (
  id uuid PRIMARY KEY,
  name text NOT NULL,
  state text,           -- Bundesland
  zip_codes text[],     -- zugehörige PLZ
  is_active boolean DEFAULT true,
  logo_url text,
  primary_color text,
  secondary_color text,
  created_at timestamptz
)

-- citizen_profiles
citizen_profiles (
  id uuid PRIMARY KEY REFERENCES auth.users(id),
  municipality_id uuid REFERENCES municipalities(id),
  first_name text NOT NULL,
  last_name text NOT NULL,
  date_of_birth date NOT NULL,
  street text NOT NULL,
  house_number text NOT NULL,
  zip_code text NOT NULL,
  city text NOT NULL,
  phone text,
  created_at timestamptz,
  updated_at timestamptz,
  deleted_at timestamptz  -- Soft Delete
)
```

## Screens (Mobile App)
1. `OnboardingWelcome` — Splash mit App-Name + "Jetzt starten"
2. `GemeindeAuswahl` — Suchfeld + Liste der Gemeinden
3. `Registrierung` — E-Mail/Google/Apple Buttons
4. `EmailRegistrierung` — Formular E-Mail + Passwort
5. `EmailVerifizierung` — Hinweis "E-Mail bestätigen"
6. `BuergerprofilAnlegen` — Profilformular

## Technische Hinweise
- Supabase Auth für OAuth + E-Mail
- Expo `expo-secure-store` für Token-Speicherung
- `municipality_id` in Supabase Auth user metadata speichern
- RLS: `citizen_profiles` nur eigene Zeile lesbar/schreibbar

## Definition of Done
- [ ] Alle Akzeptanzkriterien erfüllt
- [ ] RLS-Policies für `citizen_profiles` implementiert
- [ ] E2E-Test: Kompletter Onboarding-Flow durchlaufen
- [ ] DSGVO-Checkbox funktioniert + wird gespeichert
- [ ] Apple Sign-In auf echtem iOS-Gerät getestet
