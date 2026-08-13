# Jass-Liga Schällebur – neue Seitenstruktur

## Was neu ist

Die App ist neu auf **10 einzelne HTML-Dateien** aufgeteilt statt einer riesigen Index-Datei. Alle Dateien liegen im selben Ordner nebeneinander (Vercel: einfach alle Files ins Projekt-Root, `index.html` wird automatisch zur Startseite).

| Datei | Bereich | Hinweis |
|---|---|---|
| `index.html` | Startseite / Hub | Übersicht mit Links zu allen Bereichen |
| `statistik.html` | Statistik | ehemals Tab "Statistik" |
| `match.html` | Match / "Jasstafel" | ehemals Tab "Match" – funktionsgleich, jetzt eigenständig ladbar |
| `members.html` | Mitglieder | ehemals Tab "Member" |
| `admin.html` | Admin | ehemals Tab "Admin" |
| `termine.html` | **Termine** | ehemals Tab "Events" (Jassrunden planen, zu-/absagen, Kalender/Email) – **umbenannt**, damit es nicht mit "Jass-Events" kollidiert |
| `events.html` | **Jass-Events** (Übersicht) | NEU – ersetzt "Generationen Jass": Liste laufender/offener/vergangener Turniere, Event erstellen |
| `event.html?id=…` | Jass-Event Detail | NEU – Anmeldung, Regeln, Team-Auslosung, Admin-Steuerung |
| `bracket.html?id=…` | Turnierbaum (live) | NEU – live einsehbar, Resultate direkt im Match erfassen |
| `freispiel.html` | Freispiel/Sonderspiel | NEU – freie Erfassung ohne Turnier, auch als Gast, Regeln frei wählbar |

**Wichtig:** Der alte "Events"-Tab (Jassrunden mit Kalender-Einladung) und die neuen "Jass-Events" (Turniere) sind zwei verschiedene Dinge und bewusst getrennt geblieben – sonst wäre es verwirrend geworden. Falls euch die Bezeichnung "Termine" nicht gefällt, sagt Bescheid, das lässt sich einfach in der Navigation (`nav_new`-Block, `NAV_ITEMS`) umbenennen.

## Beantwortung eurer Punkte

1. **Aufteilung** – erledigt, 10 Seiten statt 1.
2. **"Generationen Jass" → "Jass-Events"** – erledigt, jetzt für beliebig viele Turniere statt nur eines.
3. **Registrierung per Link + Admin-Erfassung + eigener Turnierbaum pro Event** – erledigt (`event.html?id=…`, `bracket.html?id=…`).
4. **Historie** – erledigt, Events mit Status "Beendet" erscheinen automatisch in der Historie auf `events.html`.
5. **Turnierbaum + Events als eigener Bereich, unabhängig von den Mitglieder-Seiten** – erledigt: `events.html`, `event.html`, `bracket.html`, `freispiel.html` laden nicht den ganzen Mitglieder-Code, sondern nur was sie brauchen.
6. **Freispiele für Gäste** – `freispiel.html`, kein Login nötig.
7. **Konfigurierbare Zählweise pro Event** – beim Event-Erstellen wählbar: "Klassisch" (Eichel/Rose 1×, Schälle/Schilte 2×, Obenabe/Undeufe 3× – das bisherige Schällebur-Standard), "Einfach" (alle 1×, Obenabe/Undeufe 2×), "Alles gleich" (alle 1×) oder frei editierbar. **Hinweis:** Es gibt keine einzige offizielle gesamtschweizerische Multiplikator-Tabelle, das ist Klub-/Verbandskonvention – darum die freie Wahl statt einer Vorgabe.
8. **Live-Eintragen im Turnierbaum** – jedes Match im Turnierbaum lässt sich anklicken, öffnet eine Mini-Jasstafel mit den für dieses Event geltenden Regeln. Punkte werden rundenweise gespeichert und sind für alle sofort live sichtbar (Firestore `onSnapshot`). Berechtigt zum Eintragen sind Admins sowie eingeloggte Mitglieder, die selbst in einem der beiden Teams stehen.

## Neue Firestore-Collections

Bewusst neue, eigene Collections – damit nichts mit den bestehenden `jass_events` (= Termine) kollidiert:

- `jass_tourneys` – die Jass-Events selbst
- `jass_tourney_registrations` – Anmeldungen pro Event
- `jass_tourney_brackets` – ein Dokument pro Event (Turnierbaum inkl. laufender Resultate)
- `jass_free_games` – Freispiele

Die alten `jass_genjass_reg` / `jass_bracket` Collections werden von der neuen Struktur nicht mehr verwendet (alte Daten von "Generationen Jass" wurden nicht automatisch migriert – sagt Bescheid, falls ihr die alten Anmeldungen/den alten Baum noch braucht, das lässt sich nachträglich importieren).

## Firestore Security Rules (Vorschlag)

Da sich bei euch auch nicht eingeloggte Gäste registrieren und Freispiele erfassen können sollen, braucht es hierfür offene, aber eingeschränkte Schreibrechte. Ungetesteter Vorschlag zum Ergänzen eurer bestehenden `firestore.rules` – bitte vor dem Deployen selbst in der Firebase Console gegenprüfen:

```
match /jass_tourneys/{id} {
  allow read: if true;
  allow write: if request.auth != null; // Feinere Admin-Prüfung nach Bedarf ergänzen
}
match /jass_tourney_registrations/{id} {
  allow read: if true;
  allow create: if true; // auch Gäste ohne Login dürfen sich anmelden
  allow update, delete: if request.auth != null;
}
match /jass_tourney_brackets/{id} {
  allow read: if true;
  allow write: if request.auth != null; // Eintragen nur eingeloggt
}
match /jass_free_games/{id} {
  allow read: if true;
  allow create: if true; // auch Gäste dürfen Freispiele erfassen
  allow update, delete: if request.auth != null;
}
```

## Bekannte Einschränkungen / offene Punkte

- Die neuen Seiten (`events`, `event`, `bracket`, `freispiel`, `index`) sind nur auf Deutsch (kein DE/EN-Umschalter wie bei den bestehenden Seiten) – lässt sich bei Bedarf nachrüsten.
- Der Turnierbaum unterstützt 4, 8 oder 16 Teams (Doppel-K.O., gleiche Logik wie zuvor).
- Ich konnte die Seiten nicht in einem echten Browser gegentesten (kein Firebase-Zugriff aus dieser Umgebung) – der komplette JavaScript-Code wurde aber mit Node.js auf Syntaxfehler geprüft und alle Funktionsaufrufe gegengecheckt. Bitte nach dem Hochladen auf Vercel einmal alle Bereiche kurz durchklicken, bevor ihr es an den Klub kommuniziert.
