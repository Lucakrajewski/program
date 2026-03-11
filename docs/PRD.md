# Product Requirements Document — ORTSHELD

## Vision
ORTSHELD ist eine skalierbare Multi-Tenant-SaaS-Plattform für deutsche Städte und Gemeinden. Ziel ist es, die digitale Verbindung zwischen Gemeinden und ihren Bürgern zu stärken: Informationen, Services und Bürgerbeteiligung — alles in einer App.

**Motto:** _Deine Gemeinde. Digital. Erleben._

## Target Users

### Bürger
- Einwohner einer Gemeinde, die schnell auf lokale Informationen und Services zugreifen wollen
- Schmerzpunkte: lange Wartezeiten beim Rathaus, fehlende Transparenz bei Meldungen, schlechte Erreichbarkeit der Verwaltung
- Nutzung: Expo (React Native) App — iOS & Android

### Gemeinde-Verwaltung
- Mitarbeiter der Gemeindeverwaltung (Admin, Redakteur, Stadtrat/Verwaltung)
- Schmerzpunkte: ineffiziente Kommunikation, manuelle Prozesse, fehlende digitale Tools
- Nutzung: Web-Admin-Portal (Next.js)

### Touristen (Phase 2)
- Besucher der Gemeinde, die lokale Highlights entdecken wollen

## Tech Stack
- **Mobile App:** Expo (React Native) — iOS & Android
- **Admin-Portal:** Next.js 16 (App Router), TypeScript
- **Backend:** Supabase (PostgreSQL + Auth + RLS + Storage)
- **Styling:** Tailwind CSS + shadcn/ui (Admin-Portal)
- **Deployment:** Vercel (Admin-Portal), Expo EAS (Mobile App)
- **Validierung:** Zod + react-hook-form

## Core Features (Roadmap)

| Priority | Feature | Status |
|----------|---------|--------|
| P0 (MVP) | Onboarding + Gemeinde-Auswahl | Planned |
| P0 (MVP) | Authentifizierung (E-Mail, Google, Apple) | Planned |
| P0 (MVP) | Bürgerprofil | Planned |
| P0 (MVP) | News & Events | Planned |
| P0 (MVP) | Mängelmelder (Foto + GPS + Status) | Planned |
| P0 (MVP) | Rathaus-Chat (kategorie-basiert) | Planned |
| P0 (MVP) | Gemeinde Admin-Portal (Grundfunktionen) | Planned |
| P1 | Terminbuchung | Planned |
| P1 | Müllkalender (adressbasiert) | Planned |
| P1 | Bürgerbeteiligung (Umfragen, Ideen) | Planned |
| P1 | Lokale Wirtschaft (Branchenverzeichnis) | Planned |
| P1 | Jobs & Ehrenamt | Planned |
| P1 | Community (Schwarzes Brett, Nachbarschaftshilfe) | Planned |
| P1 | Nachhaltigkeit & Infrastruktur | Planned |
| P2 | Tourismus-Modul (Sehenswürdigkeiten, Karte, Restaurants) | Planned |
| P2 | Gamification (QR-Scan, Verlosungen, Gutscheine) | Planned |
| P2 | Simpler Regel-Bot (Öffnungszeiten etc.) | Planned |
| P3 | KI-Modul (RAG, Antwortvorschläge, Wissensdatenbank) | Planned |

## Multi-Tenant-Architektur
- Jede Gemeinde besitzt eine eindeutige `municipality_id`
- Alle Daten sind mandantengetrennt via Supabase RLS
- Feature-Flags pro Gemeinde (Module aktivieren/deaktivieren)
- White-Label: Logo, Farben, Texte pro Gemeinde konfigurierbar
- Kein Cross-Tenant-Zugriff möglich

## Rollenmodell

| Rolle | Bereich | Rechte |
|-------|---------|--------|
| Bürger | Mobile App | Eigene Daten, Meldungen, Chat, Umfragen |
| Redakteur | Admin-Portal | Inhalte (News, Events, Wirtschaft) verwalten |
| Verwaltung/Stadtrat | Admin-Portal | Umfragen starten, Mängel bearbeiten, Chat antworten |
| Admin | Admin-Portal | Vollzugriff, Feature-Flags, Rollen, Branding |

## Success Metrics
- Anzahl aktiver Gemeinden (Ziel MVP: 3 Pilot-Gemeinden)
- Monatlich aktive Nutzer pro Gemeinde
- Mängelmelder-Abschlussquote (Eingegangen → Erledigt)
- Chat-Antwortzeit der Verwaltung
- Umfragebeteiligung in %
- App Store Bewertung ≥ 4.0

## Constraints
- DSGVO-Konformität (Deutschland) — zwingend
- Mandantenisolierung — keine Kompromisse
- Mobile App: iOS 15+ und Android 10+
- Supabase als primäre Backend-Infrastruktur
- Keine direkte DB-Abfrage durch KI (Phase 3: RAG only)

## Non-Goals (MVP)
- KI-Modul (Phase 3)
- Gamification (Phase 2)
- Live-GPS-Tracking für Busse (langfristig)
- Bezahlsystem / E-Commerce
- Eigene Kartendaten (OpenStreetMap/Google Maps API wird genutzt)
- Mehrsprachigkeit (initial nur Deutsch)

---

_Letzte Aktualisierung: 2026-03-11_
