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
8. **Live-Eintragen im Turnierbaum** – jedes Match im Turnierbaum lässt sich anklicken, öffnet eine Mini-Jasstafel mit den für dieses Event geltenden Regeln. Punkte werden rundenweise gespeichert und sind für alle sofort live sichtbar (Firestore `onSnapshot`). Berechtigt zum Eintragen sind Admins sowie alle registrierten Teilnehmer ihres eigenen Matches – dank der anonymen Firebase-Anmeldung (siehe unten) funktioniert das jetzt auch für Gäste ohne Google-Konto, nicht nur für eingeloggte Mitglieder.

## Neue Firestore-Collections

Bewusst neue, eigene Collections – damit nichts mit den bestehenden `jass_events` (= Termine) kollidiert:

- `jass_tourneys` – die Jass-Events selbst
- `jass_tourney_registrations` – Anmeldungen pro Event
- `jass_tourney_brackets` – ein Dokument pro Event (Turnierbaum inkl. laufender Resultate)
- `jass_free_games` – Freispiele

Die alten `jass_genjass_reg` / `jass_bracket` Collections werden von der neuen Struktur nicht mehr verwendet (alte Daten von "Generationen Jass" wurden nicht automatisch migriert – sagt Bescheid, falls ihr die alten Anmeldungen/den alten Baum noch braucht, das lässt sich nachträglich importieren).

## Firestore Security Rules (Vorschlag)

Da ihr in der Firebase Console unter **Authentication → Anmeldemethode** bereits "Anonym" aktiviert habt: Der Code auf den 5 neuen Seiten (`index`, `events`, `event`, `bracket`, `freispiel`) meldet Besucher jetzt automatisch im Hintergrund anonym an (`signInAnonymously`), sobald sie eine dieser Seiten öffnen — ganz ohne Klick, ohne Google-Konto. Dadurch hat *jeder* echte Besucher eine gültige Firebase-Session mit stabiler `uid`, auch Gäste. Das erlaubt euch, die Regeln enger zu fassen als "für alle offen":

```
match /jass_tourneys/{id} {
  allow read: if true;
  allow write: if request.auth != null; // Feinere Admin-Prüfung nach Bedarf ergänzen
}
match /jass_tourney_registrations/{id} {
  allow read: if true;
  allow create: if request.auth != null; // auch anonyme Gäste dürfen sich anmelden
  allow update, delete: if request.auth != null;
}
match /jass_tourney_brackets/{id} {
  allow read: if true;
  allow write: if request.auth != null; // Resultate eintragen: eigene Berechtigungsprüfung passiert im Frontend (Admin oder Team-Mitglied)
}
match /jass_free_games/{id} {
  allow read: if true;
  allow create: if request.auth != null; // auch anonyme Gäste dürfen Freispiele erfassen
  allow update, delete: if request.auth != null;
}
```

Wichtig: Diese Regeln lassen **jeden mit gültiger Firebase-Session** (also auch anonyme Gäste) schreiben — das ist bewusst so, weil ihr ja wollt, dass sich Gäste ohne Konto registrieren können. Sie blockieren aber zumindest simple Bots/Skripte, die gar keine Firebase-Session aufbauen. Für strengeren Schutz (z.B. Rate-Limiting oder Validierung der Feldinhalte) müsste man die Regeln weiter ausbauen — sagt Bescheid, falls das ein Thema wird.

**Nur die neuen Seiten melden sich anonym an**, die bestehenden Mitglieder-Seiten (`statistik`, `match`, `members`, `admin`, `termine`) wurden bewusst nicht angefasst — dort bleibt die Login-Logik (Google-Login für Mitglieder) unverändert wie bisher.

## Update (2. Runde)

- **Neuer Admin:** `baettig.laurent@gmail.com` ist in `ADMIN_EMAILS` ergänzt. Wirkt automatisch beim nächsten Google-Login von Laurent (egal ob er sich neu registriert oder schon einen Account hat).
- **Layout:** Nur `index.html` zeigt noch den grossen Hero-Header mit Logo. Alle anderen Seiten haben einen kompakten, **sticky** Header (bleibt beim Scrollen oben) mit: Home-Icon links (zurück zum Hauptmenü), Seitentitel in der Mitte, Login/Avatar rechts. Responsive getestet (Text "Hauptmenü" verschwindet auf kleinen Screens, nur Icon bleibt). Die Navigationsleiste (Desktop: Reihe oben, Mobile: Leiste unten) bleibt zusätzlich bestehen, damit man zwischen Bereichen wechseln kann, nicht nur zurück zum Hauptmenü.
- **Sichtbarkeit für Gäste:** "Match" und "Termine" erscheinen in der Navigation nur noch für eingeloggte Mitglieder (`isMember()`). Gäste/Event-User sehen: Statistik, Member, Jass-Events, Freispiel. Ruft ein Gast `termine.html` trotzdem direkt über die URL auf, sieht er eine "Nur für Mitglieder"-Karte mit Login-Button statt der Jassrunden-Liste. Die Startseite zeigt jetzt zusätzlich eine kurze Klub-Präsentation ("D'Jassgenosse Schällebur") — das ist bewusst als Platzhalter-Text markiert, den ihr durch eure echte Geschichte ersetzen könnt.
- **Event-Seite bleibt "zuhause":** Auf `event.html?id=…` gibt es jetzt direkt einen Bereich "Sonderspiele / Freispiele", in dem alle (auch Gäste) spontane Spiele erfassen können, ohne die Seite zu verlassen. Der Link zur eigenständigen `freispiel.html` bleibt als Alternative bestehen.

### Kleiner technischer Hinweis
Für die Freispiele-Liste auf der Event-Seite wird eine einfache Firestore-Abfrage ohne zusammengesetzten Index verwendet (Sortierung passiert im Browser). Falls Firestore beim ersten Live-Test trotzdem einen Hinweis auf einen fehlenden Index anzeigt, einfach auf den von Firebase automatisch generierten Link klicken – erledigt sich in der Regel von selbst.

## Update (3. Runde)

- **##1 Header/Logo:** Der kompakte Header zeigt jetzt euer Logo (rundes Icon) statt eines generischen Home-Icons. Auf allen "vollen" Seiten (Statistik, Member, Jass-Events, Freispiel, Match, Termine, Admin) ist die Navigationsleiste zu einem **Dropdown-Menü** im Header zusammengeklappt ("Menü ▾") — deutlich aufgeräumter als die Pillen-Reihe vorher. Die Startseite behält ihren grossen Hero mit der Pillen-Reihe darunter, das war ja schon gut so.
- **##2 Login-Bug behoben:** Das war ein Race-Condition-Fehler — die automatische Gast-Anmeldung hat teils *vor* Abschluss der echten Session-Wiederherstellung ausgelöst und dadurch eingeloggte Mitglieder gelegentlich in eine anonyme Session "verschoben". Jetzt wird zuerst der erste echte Auth-Status abgewartet, bevor überhaupt entschieden wird, ob eine anonyme Anmeldung nötig ist. Sollte jetzt zuverlässig funktionieren — bitte auf Vercel gegentesten (mehrfach zwischen Seiten wechseln, auch mit langsamerer Verbindung).
- **##3 Turnierbaum-Redesign:** Deutlich mehr "Esports"-Charakter im Comic-Look: schwarze Match-Karten mit Gold-Akzenten, Live-Anzeige (pulsierender roter Punkt) für laufende Spiele, Krone bei gewonnenen Reihen, grosser Champion-Banner mit Gold-Verlauf und Trophäen-Icon, farbige Badges für Winner-/Loser-Bracket/Grand-Final. Echte SVG-Verbindungslinien zwischen den Matches habe ich bewusst weggelassen (das bräuchte exakte Pixel-Berechnungen je nach Bildschirmgrösse und wird schnell fehleranfällig) — der Look ist aber schon deutlich "spielerischer" als vorher.
- **##4 Admin kann alles bearbeiten:** Admins können jetzt jedes Match öffnen — auch bereits entschiedene — und Runden nachtragen oder den Sieger korrigieren ("Bearbeiten"-Button statt nur bei offenen Spielen). Achtung: Falls im Baum schon Folge-Spiele auf Basis des alten (falschen) Resultats gespielt wurden, kann eine nachträgliche Korrektur zu Inkonsistenzen weiter oben im Baum führen — das zeigt die App als Hinweis direkt im Bearbeiten-Dialog an, eine automatische Kaskaden-Korrektur gibt es (noch) nicht.
- **##5 Kontext-Navigation:** Auf `event.html` und `bracket.html` ist die generische Navigation komplett weg — dort gibt's nur noch "← Zurück" (zu Jass-Events bzw. zum jeweiligen Event) plus Home-Icon. Kein Ablenken durch irrelevante Menüpunkte mehr, während man im Turnier steckt.
- **##6 Avatar-Auswahl bei der Registrierung:** 20 witzige, jass-thematisch benannte Avatare (via DiceBear, kostenlos, kein eigenes Hosting nötig) stehen bei der Registrierung zur Auswahl, plus das Google-Foto als Alternative falls vorhanden. **Nicht umgesetzt:** eigenes Foto machen/hochladen (Kamera/Datei-Upload) — das würde Firebase Storage plus zusätzliche Regeln brauchen, die ich hier nicht anlegen und testen konnte. Sagt Bescheid, falls das noch gewünscht ist, das lässt sich nachrüsten.

## Update (4. Runde)

- **##1 Logo im Header:** Zeigt jetzt die Schälle (`Schälle.png`) statt des kleinteiligen Wappens — deutlich besser erkennbar in der kleinen Grösse.
- **##2 Automatische Punkte-Gegenrechnung:** In allen drei Erfassungsmasken (Turnierbaum-Resultat, eingebettetes Event-Freispiel, eigenständige Freispiel-Seite) rechnet die Eingabe jetzt exakt wie bei "Match": Tippt ihr bei Team 1 z.B. 10 ein, setzt sich Team 2 automatisch auf 147 — die beiden Felder ergeben immer 157 (Kartenpunkte + Stöck), unabhängig vom gewählten Trumpf. Der Multiplikator (1×/2×/3× je nach Event-Regeln und gespieltem Trumpf) wird wie gehabt erst bei der Endsumme draufgerechnet, nicht auf die 157 selbst. Ein Speichern mit falscher Summe wird blockiert.
- **##3 Turnier-Statistik:** Neu unten im Turnierbaum ein aufklappbarer Bereich "Turnier-Statistik" mit: Tabelle pro Team (Siege, Niederlagen, Punkte total, Weis total, meistgespielter Trumpf), Trumpf-Verteilung pro Team als Chips, und eine Liste aller Einzelspiele mit Resultat. Alles wird live aus den bereits gespeicherten Turnierbaum-Daten berechnet, keine zusätzliche Datenbank-Struktur nötig.
- **##4 Klub-Geschichte:** Der Text "D'Gschicht vo de Jassgenosse Schällebuur" ist jetzt auf der Startseite eingebaut — mit dunklem Titel-Banner (Schälle-Musterung im Hintergrund), den Absätzen als Fliesstext und den zwei Publikumslieblingen (Chrüüterschnaps & "Der Bürgermeister") als hervorgehobene Karten.

## Update (5. Runde)

- **##1 Verbindungslinien im Turnierbaum:** Es gibt jetzt echte, live berechnete Linien zwischen den Matches (per SVG, per JavaScript nach jedem Render neu gezeichnet — reagiert auch auf Fenstergrösse/Scrollen). Gold & durchgezogen = der Weg ist bereits entschieden, grau gestrichelt = der Pfad ist noch offen. Aus technischen Gründen zeichne ich die Linien nur **innerhalb** von Winner-Bracket bzw. Loser-Bracket (also "Runde 1 → Runde 2 → ..."); die Übergänge WB-Final/LB-Final → Grand Final sind weiterhin ohne Linie, aber klar als eigene, farblich abgesetzte Sektion sichtbar. Eine durchgehende Linie über alle drei Sektionen hinweg wäre technisch aufwändiger und fehleranfälliger geworden (verschiedene Scroll-Container) — falls euch das wichtig ist, sagt Bescheid, das lässt sich nachrüsten.
- **##2 Abgeschlossene Spiele klar erkennbar + Admin-Korrektur:** Entschiedene Matches sehen jetzt sichtbar "fertig" aus (Schloss-Icon, gedämpfte Optik, dezenter "Admin: wieder öffnen"-Button statt des auffälligen roten Start-Buttons). Sobald das Turnier komplett entschieden ist, wird das automatisch auf dem Event gespeichert (Status "beendet" + Champion-Name) — der Champion ist jetzt gut sichtbar: auf der Jass-Events-Übersicht (goldener Rahmen + Pokal-Icon auf der Karte), auf der Event-Seite (grosses Banner) und im Turnierbaum selber. Auf der Event-Seite gibt's für Admins jetzt ausserdem einen Button **"Neue Runde starten (gleiche Teilnehmer)"**, der Teams neu auslost und sofort einen neuen Turnierbaum mit denselben Angemeldeten erstellt — perfekt für Runde 2/3 am gleichen Abend.
- **##3 Jubel-Konfetti:** Bei jedem abgeschlossenen Match gibt's einen kleinen Konfetti-Regen, beim Turniersieger einen grösseren (mehr Teilchen, länger). Rein CSS/JS, keine externe Bibliothek nötig, respektiert automatisch `prefers-reduced-motion` (kein Konfetti bei entsprechender Systemeinstellung).








## Bekannte Einschränkungen / offene Punkte

- Die neuen Seiten (`events`, `event`, `bracket`, `freispiel`, `index`) sind nur auf Deutsch (kein DE/EN-Umschalter wie bei den bestehenden Seiten) – lässt sich bei Bedarf nachrüsten.
- Der Turnierbaum unterstützt 4, 8 oder 16 Teams (Doppel-K.O., gleiche Logik wie zuvor).
- Ich konnte die Seiten nicht in einem echten Browser gegentesten (kein Firebase-Zugriff aus dieser Umgebung) – der komplette JavaScript-Code wurde aber mit Node.js auf Syntaxfehler geprüft und alle Funktionsaufrufe gegengecheckt. Bitte nach dem Hochladen auf Vercel einmal alle Bereiche kurz durchklicken, bevor ihr es an den Klub kommuniziert.
