# ORTSHELD — Technische Architektur

> **Zielgruppe:** Product Owner, Entwickler, zukünftige Team-Mitglieder
> **Letzte Aktualisierung:** 2026-03-11
> **Status:** Genehmigt — Basis für alle Entwicklungsarbeiten

---

## 1. Strategische Entscheidung: Monorepo

### Was ist ein Monorepo?
Ein Monorepo bedeutet: **alle Teile des Projekts leben in einem einzigen Git-Repository** — die mobile App, das Web-Admin-Portal und alle gemeinsamen Bausteine. Das ist der Standard bei erfolgreichen Produkten wie Airbnb, Spotify und Linear.

### Warum Monorepo und nicht getrennte Repos?

| Kriterium | Monorepo ✅ | Getrennte Repos ❌ |
|-----------|-------------|-------------------|
| Geteilte Typen (Bürger, Gemeinde, Meldung) | Eine Datei, überall gültig | Doppelt gepflegt → Fehler |
| Validierungsregeln (z.B. PLZ-Format) | Einmal schreiben, überall nutzen | Immer wieder kopieren |
| Bug in Datenbankabfrage | Einmal fixen | In 2 Repos fixen |
| Neue Gemeinde onboarden | Ein Deployment | Koordination über 2 Repos |
| Neue Entwicklerin einarbeiten | Ein `git clone`, fertig | 2 Repos, 2 Setups |
| CI/CD Pipeline | Eine Pipeline | Zwei Pipelines synchronisieren |

**Fazit:** Für ein SaaS-Unternehmen, das schnell skalieren will, ist das Monorepo-Modell der einzig sinnvolle Weg.

### Technologie: Turborepo
Turborepo ist das führende Tool für Monorepos in der JavaScript/TypeScript-Welt. Es sorgt dafür, dass nur geänderte Teile neu gebaut werden (intelligentes Caching) — was Deployment-Zeiten um bis zu 80% reduziert.

---

## 2. Gesamt-Systemarchitektur

```
ortsheld/                          ← Ein Git-Repository
│
├── apps/
│   ├── mobile/                    ← Bürger-App (Expo / React Native)
│   │   iOS App Store + Google Play Store
│   │
│   └── web/                       ← Gemeinde Admin-Portal (Next.js)
│       Browser-basiert, kein App Store nötig
│
├── packages/
│   ├── shared/                    ← Geteilte Grundbausteine
│   │   ├── types/                 Alle Datenstrukturen (Bürger, Gemeinde, Meldung...)
│   │   ├── validators/            Validierungsregeln (PLZ, E-Mail, Telefon...)
│   │   └── constants/             Kategorien, Status-Werte, Feature-Keys
│   │
│   ├── api-client/                ← Datenbankabfragen (einmal schreiben, überall nutzen)
│   │   Alle Abfragen an Supabase zentral — Mobile + Web nutzen dieselben Funktionen
│   │
│   └── config/                    ← Projekt-Konfigurationen
│       TypeScript, ESLint, Tailwind — einheitlich für alle Apps
│
└── supabase/                      ← Datenbank-Definitionen
    ├── migrations/                Datenbank-Änderungen (versioniert, rückführbar)
    ├── functions/                 Server-seitige Logik (Bot, Push-Notifications, E-Mails)
    └── seed/                      Testdaten für Entwicklung
```

### Datenfluss — So kommunizieren die Teile

```
┌─────────────────────┐     ┌─────────────────────┐
│   Bürger-App        │     │  Admin-Portal        │
│   (Expo / iOS+And)  │     │  (Next.js / Browser) │
└────────┬────────────┘     └──────────┬───────────┘
         │                             │
         │  Beide nutzen packages/api-client
         │                             │
         └──────────────┬──────────────┘
                        │
              ┌─────────▼──────────┐
              │     SUPABASE       │
              │                    │
              │  ┌──────────────┐  │
              │  │  PostgreSQL  │  │  ← Alle Daten
              │  │  (Datenbank) │  │
              │  └──────────────┘  │
              │  ┌──────────────┐  │
              │  │     Auth     │  │  ← Login, Rollen
              │  └──────────────┘  │
              │  ┌──────────────┐  │
              │  │   Storage    │  │  ← Fotos, PDFs, Logos
              │  └──────────────┘  │
              │  ┌──────────────┐  │
              │  │   Realtime   │  │  ← Chat (Live-Updates)
              │  └──────────────┘  │
              │  ┌──────────────┐  │
              │  │Edge Functions│  │  ← Bot, E-Mails, Push
              │  └──────────────┘  │
              └────────────────────┘
```

---

## 3. Multi-Tenant-Strategie: Wie Gemeinden isoliert werden

### Das Prinzip
Alle Gemeinden teilen **eine Datenbank** — aber jede Zeile in jeder Tabelle ist mit einer `municipality_id` gestempelt. Die Datenbank-Sicherheitsregeln (Row Level Security / RLS) stellen sicher, dass **niemand auf fremde Gemeindedaten zugreifen kann** — nicht durch Bugs, nicht durch Hackerversuche.

```
Datenbank (eine einzige)
│
├── Gemeinde A (Musterstadt)     municipality_id = "aaa-111"
│   ├── News: [nur Musterstadt-News]
│   ├── Meldungen: [nur Musterstadt-Meldungen]
│   └── Chat: [nur Musterstadt-Chats]
│
├── Gemeinde B (Beispieldorf)    municipality_id = "bbb-222"
│   ├── News: [nur Beispieldorf-News]
│   ├── Meldungen: [nur Beispieldorf-Meldungen]
│   └── Chat: [nur Beispieldorf-Chats]
│
└── [RLS-Regel]: Jede Anfrage sieht NUR die eigene municipality_id
```

### Warum eine gemeinsame Datenbank (nicht eine pro Gemeinde)?
- **Kosten:** 1 Datenbank = niedrige Fixkosten. 500 Gemeinden × eigene Datenbank = nicht skalierbar
- **Wartung:** Ein Schema-Update = alle Gemeinden aktualisiert. Kein manuelles Update in 500 DBs
- **Onboarding:** Neue Gemeinde = eine Zeile in der `municipalities`-Tabelle einfügen. Fertig

### Neue Gemeinde anlegen — So einfach ist das Onboarding
```
Schritt 1: Gemeinde in municipalities-Tabelle anlegen (Name, PLZ, Logo, Farben)
Schritt 2: Feature-Flags setzen (welche Module sind gebucht?)
Schritt 3: Admin-Account erstellen und einladen
Schritt 4: Gemeinde ist in der App für Bürger sichtbar
```
**Das dauert < 5 Minuten.** Kein technischer Aufwand pro Gemeinde.

---

## 4. Feature-Flag-System

Jede Gemeinde kann Module aktivieren/deaktivieren — ohne App-Update:

```
Gemeinde A (Großstadt — Premium-Paket)     Gemeinde B (Kleinst-Gemeinde — Basis)
─────────────────────────────────          ──────────────────────────────────────
✅ News & Events                            ✅ News & Events
✅ Mängelmelder                             ✅ Mängelmelder
✅ Rathaus-Chat                             ❌ Rathaus-Chat (nicht gebucht)
✅ Terminbuchung                            ❌ Terminbuchung (nicht gebucht)
✅ Müllkalender                             ✅ Müllkalender
✅ Tourismus-Modul                          ❌ Tourismus-Modul
✅ KI-Assistent                             ❌ KI-Assistent
```

**Geschäftlicher Vorteil:** Basis-, Standard- und Premium-Pakete verkaufbar. Gemeinden upgraden innerhalb der App — kein App-Store-Update nötig.

---

## 5. Bürger-App: Expo (React Native)

### Was ist Expo?
Expo ist das führende Framework für mobile Apps mit React. Man schreibt **einmal Code** → läuft auf iOS und Android. Genutzt von Shopify, Discord, und tausenden anderen Apps.

### Warum Expo (und nicht Flutter oder native Entwicklung)?
| | Expo | Flutter | Native (Swift/Kotlin) |
|--|------|---------|----------------------|
| Code teilen mit Web-Portal | ✅ (selbes TypeScript) | ❌ (Dart) | ❌ |
| Typen teilen (packages/shared) | ✅ | ❌ | ❌ |
| OTA-Updates (ohne App-Store) | ✅ | ❌ | ❌ |
| Entwicklungsgeschwindigkeit | Schnell | Mittel | Langsam |
| Entwickler-Pool | Groß (React-Entwickler) | Mittel | Klein + teuer |

### OTA-Updates — Der Killer-Vorteil für SaaS
Mit Expo können **Bugfixes und Inhaltsänderungen** direkt auf das Handy der Nutzer gespielt werden — **ohne Wartezeit durch App-Store-Review** (die 1-3 Tage dauern kann).

```
Ohne OTA:  Bug entdeckt → Fix schreiben → App Store einreichen → 1-3 Tage warten → Nutzer müssen manuell updaten
Mit OTA:   Bug entdeckt → Fix schreiben → Deploy → Nutzer sehen Fix beim nächsten App-Start
```

### App-Struktur (Navigation)

```
Bürger-App
│
├── Onboarding (nur beim ersten Start)
│   ├── Gemeinde auswählen
│   ├── Registrieren / Einloggen
│   └── Bürgerprofil anlegen
│
└── Haupt-App (Tab-Navigation)
    ├── 🏠 Start          → News-Feed + aktuelle Events
    ├── 🗺 Entdecken      → Karte + lokale Angebote [Feature-Flag]
    ├── ⚠️ Melden         → Mängelmelder
    ├── 💬 Rathaus        → Chat + Terminbuchung [Feature-Flag]
    └── 👤 Profil         → Einstellungen, Gemeinde wechseln
```

---

## 6. Admin-Portal: Next.js (Web)

### Struktur

```
Admin-Portal (Browser)
│
├── /login                    Anmeldung
│
└── /dashboard                (nach Login, geschützt)
    ├── /                     Übersicht / KPIs
    ├── /news                 News verwalten
    ├── /events               Events verwalten
    ├── /issues               Mängelmeldungen bearbeiten
    ├── /chat                 Bürger-Anfragen beantworten
    ├── /appointments         Terminverwaltung
    ├── /settings
    │   ├── /branding         Logo + Farben
    │   ├── /features         Feature-Flags
    │   ├── /team             Mitarbeiter + Rollen
    │   └── /audit            Audit-Log
    └── [System-Admin only]
        └── /municipalities   Neue Gemeinden anlegen
```

### Rollenbasierte Sichtbarkeit
```
Admin sieht:     Alle Menüpunkte + /settings komplett + /municipalities
Redakteur sieht: /news, /events
Verwaltung sieht: /issues, /chat, /appointments
```

---

## 7. Datenbank-Übersicht (Plain Language)

### Kern-Tabellen
```
municipalities        → Alle Gemeinden (Name, Logo, Farben, PLZ, aktiv?)
users                 → Supabase Auth (E-Mail, OAuth-Provider)
citizen_profiles      → Bürgerprofil (Name, Adresse, Telefon)
municipality_staff    → Verwaltungsmitarbeiter (Rolle, Gemeinde)
municipality_settings → Branding pro Gemeinde
feature_flags         → Welche Module sind pro Gemeinde aktiv?
audit_logs            → Wer hat wann was geändert?
```

### Inhalts-Tabellen
```
news_posts            → Nachrichten-Beiträge
events                → Veranstaltungen
event_registrations   → Anmeldungen zu Events
issue_reports         → Mängelmeldungen
issue_status_history  → Statusverlauf einer Meldung
chat_threads          → Chat-Anfragen
chat_messages         → Einzelne Nachrichten im Chat
chat_bot_rules        → Regelwerk für automatische Antworten
appointments          → Termine (Phase 2)
waste_schedules       → Müllkalender (Phase 2)
```

### Sicherheitsregeln (RLS) — Das Herzstück der Mandantentrennung

Jede Tabelle hat Regeln nach diesem Schema:
```
Bürger darf lesen:    Nur veröffentlichte Inhalte der eigenen Gemeinde
Bürger darf schreiben: Nur eigene Datensätze (eigene Meldungen, eigenes Profil)
Staff darf lesen:      Alle Daten der eigenen Gemeinde
Staff darf schreiben:  Alle Daten der eigenen Gemeinde
System-Admin:          Alles (nur interne Verwaltung, kein Frontend)
```

---

## 8. Serverseite: Supabase Edge Functions

Edge Functions sind kleine Server-Programme, die auf Ereignisse reagieren. Für ORTSHELD:

| Funktion | Auslöser | Was passiert |
|----------|----------|--------------|
| `send-push-notification` | Meldungsstatus ändert sich | Push an Bürger |
| `send-push-notification` | Neue Chat-Antwort | Push an Bürger/Staff |
| `bot-matcher` | Neuer Chat-Thread | Bot-Regeln prüfen, ggf. Antwort senden |
| `send-confirmation-email` | Event-Anmeldung | Bestätigungs-E-Mail |
| `send-reminder` | 24h vor Event (Scheduler) | Push-Erinnerung an angemeldete Nutzer |
| `new-municipality-setup` | Gemeinde angelegt | Default Feature-Flags + Settings anlegen |

---

## 9. Deployments & CI/CD

```
Code-Änderung in Git
        │
        ▼
GitHub Actions (automatisch)
        │
        ├── Tests laufen (alle Pakete)
        ├── TypeScript-Typen prüfen
        └── Lint prüfen
                │
         ✅ Erfolgreich?
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
   Vercel           Expo EAS
  (Web-Admin)      (Mobile App)
        │               │
        ▼               ├── OTA Update (sofort, kein App Store)
  Live in < 1 Min       └── Build → App Store / Play Store (bei Major-Updates)
```

### Umgebungen
```
development  → Lokale Entwicklung, eigene Supabase-Instanz
staging      → Testumgebung, echte Daten nur von Test-Gemeinden
production   → Live-System, alle Gemeinden
```

---

## 10. Skalierungs-Strategie

### Heute (MVP — 1-10 Gemeinden)
- Supabase Free/Pro Plan
- Vercel Hobby/Pro Plan
- Expo EAS Free Plan
- **Monatliche Infrastrukturkosten: ~50-100 €**

### Wachstum (11-100 Gemeinden)
- Supabase Pro Plan mit Add-ons
- Vercel Pro Plan
- Supabase Connection Pooling (PgBouncer) aktiv
- **Monatliche Infrastrukturkosten: ~200-500 €**

### Scale (100+ Gemeinden)
- Supabase Enterprise oder eigenes PostgreSQL auf AWS/Hetzner
- Vercel Enterprise oder eigenes Kubernetes
- CDN für Bilder (Cloudflare)
- **Monatliche Infrastrukturkosten: 1.000-3.000 €** (bei entsprechendem Umsatz)

**Wichtig:** Die Architektur ändert sich bei Skalierung **nicht**. Nur die Infrastruktur wird größer. Kein Refactoring nötig.

---

## 11. Technologie-Entscheidungen — Zusammenfassung

| Bereich | Technologie | Warum |
|---------|-------------|-------|
| Mobile App | Expo (React Native) | Code-Sharing mit Web, OTA-Updates, großer Entwickler-Pool |
| Admin-Portal | Next.js 16 | Server-Performance, SEO, shadcn/ui, Vercel-Deployment |
| Monorepo | Turborepo | Geteilte Typen/Logik, schnelle CI/CD, einfaches Onboarding |
| Datenbank | Supabase PostgreSQL | Multi-Tenant via RLS, kosteneffizient, managed |
| Auth | Supabase Auth | E-Mail + Google + Apple out-of-the-box, DSGVO-konform |
| Dateispeicher | Supabase Storage | Integriert, CDN, Zugriffsregeln per RLS |
| Echtzeit (Chat) | Supabase Realtime | Keine eigene WebSocket-Infrastruktur nötig |
| Server-Logik | Supabase Edge Functions | Bot, Push, E-Mails — nahe an der Datenbank |
| Deployment Web | Vercel | Git-Push → automatisch live, Preview-Deployments |
| Deployment Mobile | Expo EAS | OTA-Updates, App-Store-Submission automatisiert |
| Validierung | Zod | TypeScript-first, teilen zwischen Mobile + Web |
| UI (Web) | shadcn/ui + Tailwind | Bereits vorhanden, keine Custom-Komponenten nötig |
| CI/CD | GitHub Actions | Kostenlos für Open-Source, gut integriert |

---

## 12. Pakete (Dependencies)

### packages/shared
- `zod` — Validierung der Datenstrukturen

### apps/mobile (Expo)
- `expo` — App-Framework
- `expo-router` — Navigation (file-based, wie Next.js)
- `expo-location` — GPS für Mängelmelder
- `expo-camera` — Kamera für Fotos
- `expo-notifications` — Push-Benachrichtigungen
- `expo-secure-store` — Sichere Token-Speicherung
- `react-native-maps` — Karte (Apple Maps / Google Maps)
- `@supabase/supabase-js` — Datenbankanbindung
- `@tanstack/react-query` — Datenabruf + Caching
- `react-hook-form` — Formular-Verwaltung

### apps/web (Next.js Admin-Portal)
- Alle shadcn/ui Komponenten (bereits installiert)
- `@supabase/ssr` — Server-side Supabase für Next.js
- `@tanstack/react-query` — Datenabruf + Caching
- `@tiptap/react` — Rich-Text-Editor für News/Events
- `react-hook-form` + `zod` — Formulare + Validierung
- `recharts` — Diagramme für Dashboard-KPIs
- `date-fns` — Datumsformatierung (Deutsche Locale)

---

## 13. Nicht-Ziele (Architektur-Grenzen)

Diese Entscheidungen wurden bewusst **nicht** getroffen, um das Projekt fokussiert zu halten:

- ❌ **Eigene Zahlungsabwicklung** — Stripe oder Paddle werden später evaluiert
- ❌ **Eigene Karten-Infrastruktur** — Google Maps / OpenStreetMap API wird genutzt
- ❌ **Mehrsprachigkeit (i18n)** — Initial nur Deutsch; Struktur erlaubt spätere Erweiterung
- ❌ **Eigener Auth-Server** — Supabase Auth ist ausreichend und DSGVO-konform
- ❌ **Microservices** — Monolith-first, erst bei Bedarf aufteilen

---

_Dieses Dokument wird bei größeren Architekturentscheidungen aktualisiert._
