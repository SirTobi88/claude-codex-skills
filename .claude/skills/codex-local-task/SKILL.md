---
name: codex-local-task
description: Delegiert eine klar umrissene Programmieraufgabe an Codex (neuestes Sol-Modell, effort high) – rein lokal, ohne Push und ohne PR. Codex arbeitet in einem eigenen Worktree ab dem aktuellen lokalen Stand, Claude arbeitet derweil weiter, prueft danach Diff und Tests und uebernimmt das Ergebnis per Cherry-Pick als Commit in den aktuellen Branch. Auch von Claude selbst nutzbar, wenn eine Teilaufgabe gut delegierbar ist (reine Logik, Regel aus docs/ ableitbar, mit Tests pruefbar, keine Oberflaeche, kein Netz). Aufruf z. B. /codex-local-task Preisberechnung in das Kernmodul ziehen.
argument-hint: "<Aufgabe>"
allowed-tools: Bash, Read, Write, Edit, Grep
---

# Aufgabe lokal an Codex delegieren

Aufgabe: `$ARGUMENTS` – leer → nachfragen (Nutzer) bzw. nicht aufrufen (Claude).

Unterschied zu `/codex-task`: **kein Push, kein PR, kein Zugriff auf `origin`**. Start ist
der lokale Stand, Ziel ist ein Commit im aktuellen Branch. Darum passt der Skill auch fuer
Arbeit auf ungepushten Branches und fuer selbststaendiges Arbeiten ohne Rueckfragen.

Rollen: Codex implementiert. Claude schreibt den Auftrag, arbeitet waehrenddessen an
anderen Dateien weiter, prueft das Ergebnis (Diff lesen + Tests) und uebernimmt es. Claude
ist fuer das Ergebnis verantwortlich, als haette Claude es selbst geschrieben.

`S` = Scratchpad-Verzeichnis dieser Session (nicht `/tmp`).

## 0. Passt die Aufgabe?

Gut delegierbar:
- reine Logik in einem Modul/Paket, Umziehen von Logik aus der App in ein Modul, neue
  Adapter nach vorhandenem Muster;
- Regel steht in `docs/` oder ist im Auftrag eindeutig formulierbar → Tests ableitbar;
- pruefbar mit schnellen, in sich geschlossenen Tests (Unit-/Pakettests).

Nicht delegieren (selbst machen): Plattform-Builds, Emulatoren/Simulatoren, Oberflaeche und
Designabgleich, alles mit Netz, Aufgaben, deren Loesung erst beim Bauen klar wird, und
alles, was dieselben Dateien beruehrt wie die eigene laufende Arbeit.

Hoechstens **ein** Codex-Lauf gleichzeitig (Builds im Worktree und eigene Builds
konkurrieren um Speicher und CPU).

## 1. Vorbereiten

- Stand ansehen: `git branch --show-current`, `git status --short`.
- **Start = `HEAD` des aktuellen Branchs.** Braucht die Aufgabe nicht committete
  Aenderungen: eigene Aenderungen vorher regulaer committen (Projektregeln beachten);
  fremde Aenderungen (vom Nutzer oder von Werkzeugen, z. B. generierte Projektdateien) nie
  mitnehmen – dann den Nutzer fragen oder die Aufgabe so schneiden, dass sie sie nicht
  braucht.
- Branch `codex/<kurzname>` (kebab-case), pruefen dass er lokal nicht existiert.
- Neuestes Sol-Modell (nicht hart verdrahten):
  `codex debug models 2>/dev/null | grep -o '"slug":"[^"]*-sol"' | cut -d'"' -f4 | sort -V | tail -1`
  Leer → abbrechen, auf `/codex:setup` verweisen.
- Worktree (Arbeitsbaum bleibt unberuehrt):
  `git worktree add -b codex/<name> "$S/codex-<name>" HEAD`
- **Abhaengigkeiten vorab aufloesen** – Codex hat im Sandbox kein Netz. Fuer jedes
  Modul/Paket, das die Pruefung baut, die Abhaengigkeiten vorher im Worktree holen
  (z. B. `npm ci`, `swift package resolve`, `cargo fetch`, `go mod download`).
  <!-- Anpassen: hier die konkreten Befehle fuer dein Projekt eintragen. -->
- `START=$(git rev-parse HEAD)` merken.
- Kleine Vorarbeiten fuer den Auftrag (z. B. einen Kommentar in einer Datendatei
  praezisieren) duerfen vor dem Start direkt im Worktree passieren – sie landen dann mit
  im Codex-Commit. Groessere Vorarbeit vorher regulaer committen.

## 2. Auftrag schreiben

Nach `$S/codex-<name>-prompt.md`. Anders als bei `/codex-task` ist der Auftrag ein
vollstaendiges Briefing – Codex kennt den Gespraechsverlauf nicht:

```
<aufgabe>Was genau entstehen soll, in 2–6 Saetzen. Bei Aufrufen durch den Nutzer dessen
Wortlaut uebernehmen.</aufgabe>
<kontext>
- Fundstellen: betroffene Dateien/Typen mit Pfad, Vorbild-Code ("nach dem Muster von …").
- Festlegungen: Abschnitt in docs/ (Pfad + Ueberschrift), die die Regel enthalten.
- Schnittstelle, die bestehen bleiben muss (Signaturen, die Aufrufer nutzen).
- Projektgrenzen, die hier zaehlen.
  <!-- Anpassen: z. B. erlaubte Abhaengigkeiten je Schicht, Plattformgrenzen,
  Regeln fuer Texte/Lokalisierung. -->
</kontext>
<grenzen>
- Nur diese Dateien/Ordner aendern: …
- Nicht anfassen: … (Dateien, an denen Claude parallel arbeitet, …)
</grenzen>
<regeln>
- Projektregeln aus AGENTS.md/CLAUDE.md befolgen.
  <!-- Anpassen: Kernregeln ergaenzen, z. B. Tests zuerst, Sprache der Kommentare,
  kein Code gegen docs/ (besserer Weg → erst Dokument mit Begruendung). -->
- Klein bleiben, nichts Unbeteiligtes umbauen oder umformatieren.
- Nicht committen, keine Branches, kein Netz.
- Betroffene Tests ausfuehren (Befehl laut AGENTS.md/CLAUDE.md), bis gruen. Bei
  Aenderungen an einem gemeinsamen Kernmodul auch die davon abhaengigen Module testen.
</regeln>
<abnahme>Konkrete Testfaelle bzw. Verhalten, das am Ende gelten muss.</abnahme>
<rueckgabe>
Kurz: geaenderte Dateien, neue Tests, Testergebnis (Anzahl), Annahmen und offene Punkte.
Erste Zeile: Commit-Nachricht im Stil des Repos (`git log --oneline -5`), mit Bezug aufs
Dokument, falls vorhanden.
</rueckgabe>
```

## 3. Codex starten und weiterarbeiten

Bash mit `run_in_background: true`:
```
codex exec -C "$S/codex-<name>" -s workspace-write \
  -m <sol-modell> -c model_reasoning_effort='"high"' \
  -o "$S/codex-<name>-result.md" - < "$S/codex-<name>-prompt.md" \
  > "$S/codex-<name>-log.md" 2>&1; echo "exit $?"
```
`~/.codex/config.toml` nicht aendern. Dem Nutzer eine Zeile melden (Aufgabe, Branch,
Modell). Dann **nicht warten und nicht pollen**: an Dateien ausserhalb von `<grenzen>`
weiterarbeiten, die Benachrichtigung kommt von selbst. Waehrend des Laufs keine schweren
Pruefungen (z. B. Simulator- oder End-to-End-Laeufe) im Worktree starten und die Dateien
aus `<grenzen>` im eigenen Arbeitsbaum nicht aendern.

## 4. Ergebnis pruefen

- `$S/codex-<name>-result.md` lesen; im Log `model:` und `reasoning effort:` pruefen.
- Codex' eigenes Testergebnis nicht ungeprueft uebernehmen: Im Sandbox brauchen manche
  Schnittstellen zusaetzliche Freigaben, Laeufe koennen anders ausgehen als draussen.
- `git -C "$S/codex-<name>" status --short` und `diff --stat`. Kein Diff → melden,
  aufraeumen (Schritt 6), Ende.
- **Diff lesen** (`git -C … diff`, neue Dateien ganz): Hat Codex die Aufgabe erfuellt, die
  Grenzen eingehalten, Tests zur Regel geschrieben (nicht nur angepasst)? Projektregeln
  eingehalten? Ausreisser: Dateien ausserhalb des Auftrags, Formatierungslaerm, Secrets,
  Binaries.
- **Pruefung im Worktree** nach Bereich:
  | geaendert | Pruefung |
  |---|---|
  | nur abgegrenzte Module/Pakete | Tests je betroffenem Modul und der davon abhaengigen |
  | auch App-/Integrationscode | vollstaendige Projektpruefung (z. B. Testskript laut CLAUDE.md/AGENTS.md) im Worktree – nie parallel zu einem eigenen Lauf |
- Rot oder inhaltlich falsch → nachbessern lassen, hoechstens zweimal: neuer `codex exec`
  im selben Worktree, Prompt = urspruenglicher Auftrag + Fehlerausgabe bzw. konkrete
  Beanstandung + "bitte beheben". Danach selbst fertig machen oder verwerfen und melden.
  Kleinigkeiten (Tippfehler, ein Kommentar) direkt selbst im Worktree korrigieren.

## 5. Uebernehmen (Commit + Cherry-Pick)

```
git -C "$S/codex-<name>" add -A
git -C "$S/codex-<name>" commit -m "<Commit-Titel>" -m "Umgesetzt von Codex (<sol-modell>, effort high), geprueft von Claude." -m "Co-Authored-By: …laut aktueller Attribution…"
git cherry-pick codex/<name>
```
- Vorher pruefen, dass der Arbeitsbaum keine nicht committeten Aenderungen an den Dateien
  des Commits hat (`git diff --name-only` ∩ `git diff-tree --no-commit-id --name-only -r codex/<name>`).
- Konflikt → `git cherry-pick --abort`; im Worktree `git rebase <aktueller Branch>`,
  Konflikte loesen (selbst oder Codex mit Konfliktauftrag), Pruefung wiederholen, erneut
  picken.
- Danach im eigenen Arbeitsbaum die regulaere Projektpruefung, bevor die Aufgabe als
  erledigt gilt (mehrere Uebernahmen duerfen sich einen Lauf teilen).
- Kein Push, kein PR, kein Tag – auch nicht "zum Abschluss".

## 6. Aufraeumen und melden

- Uebernommen ist der Inhalt, wenn `git cherry HEAD codex/<name>` **nichts** zeigt
  (Commit ist enthalten – hat sich `HEAD` seit dem Start nicht bewegt, spult der
  Cherry-Pick nur vor) oder nur `-`-Zeilen (gleicher Inhalt). Dann
  `git worktree remove "$S/codex-<name>"` und `git branch -D codex/<name>`.
  Nicht uebernommen (verworfen, Konflikt offen) → Worktree entfernen, Branch behalten
  und im Bericht nennen.
- Bericht knapp: Commit (Hash, Titel), geaenderte Dateien, Testergebnis, was Claude
  nachgebessert hat, offene Punkte von Codex.
