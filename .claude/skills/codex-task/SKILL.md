---
name: codex-task
description: Uebergibt eine Programmieraufgabe an Codex (neuestes Sol-Modell, Reasoning-Effort high), laesst sie in einem eigenen Worktree ab passendem Startpunkt (Standardbranch, aktueller Branch oder offener PR) umsetzen, prueft Tests und oeffnet danach einen PR. Das Review startet der Nutzer selbst. Aufruf z. B. /codex-task Validierung im Login-Formular ergaenzen.
argument-hint: "<Aufgabe>"
allowed-tools: Bash, Read, Write
---

# Aufgabe an Codex uebergeben

Aufgabe: `$ARGUMENTS` – leer → beim Nutzer nachfragen und aufhoeren.

Kein Subagent: Codex laeuft als ein Hintergrund-Befehl, Warten kostet nichts.
Selbst nichts implementieren und kein Review machen – das startet der Nutzer
(z. B. `/pr-review-codex`). Rolle hier: Auftrag formulieren, Rahmen sichern, Ergebnis abliefern.

`S` = Scratchpad-Verzeichnis dieser Session (nicht `/tmp`).

## 1. Vorbereiten

- **Startpunkt selbst entscheiden** – zuerst den Stand ansehen: `git branch --show-current`,
  `git status --short`, `git log --oneline origin/<standard>..HEAD`, offene PRs
  (`gh pr list --state open`). Dann nach der Aufgabe waehlen:
  | Aufgabe … | Start (`<start>`) | PR-Basis (`<basis>`) |
  |---|---|---|
  | unabhaengig vom laufenden Branch | `origin/<standard>` (`gh repo view --json defaultBranchRef`) | Standardbranch |
  | baut auf Commits des aktuellen Branchs bzw. eines offenen PR auf (z. B. Findings dort beheben) | dieser Branch | dieser Branch (gestapelter PR) |
  | braucht die **nicht committeten** Aenderungen im Arbeitsbaum | erst nachfragen: Nutzer committet selbst (empfohlen) oder die Aenderungen werden in den Worktree kopiert und landen dann mit im Codex-PR | wie Zeile davor |
  Ist der gewaehlte Branch lokal weiter als `origin` oder gar nicht gepusht: vom lokalen Stand
  starten; vor dem PR fragen, ob dieser Branch gepusht werden darf (sonst keinen PR, nur Branch).
  Im Zweifel zwischen zwei Zeilen kurz nachfragen statt raten.
  Wahl samt Grund in der Startmeldung an den Nutzer nennen.
- Kopieren nicht committeter Aenderungen (nur nach Zustimmung):
  `git diff HEAD --binary | git -C <worktree> apply` und unversionierte Dateien
  (`git ls-files --others --exclude-standard`) einzeln kopieren.
- Branchname aus der Aufgabe ableiten, Praefix nach Art (`feat/`, `fix/`, `chore/`), kurz, kebab-case.
  Pruefen, dass er weder lokal noch auf `origin` existiert.
- Neuestes Sol-Modell (nicht hart verdrahten):
  `codex debug models 2>/dev/null | grep -o '"slug":"[^"]*-sol"' | cut -d'"' -f4 | sort -V | tail -1`
  Leer → abbrechen, auf `/codex:setup` verweisen.
- Worktree – der Arbeitsbaum des Nutzers bleibt unberuehrt:
  ```
  git fetch origin --quiet
  git worktree add -b <branch> "$S/codex-<branch-ohne-praefix>" <start>
  ```

## 2. Auftrag schreiben

Nach `$S/codex-<name>-prompt.md`. Die Aufgabe des Nutzers woertlich uebernehmen, dazu nur
Rahmen, keine eigene Loesung. Wenn noetig, mit kurzem Blick ins Repo (z. B. `docs/`) die
betroffenen Dokumente oder Dateien als Fundstellen nennen. Gliederung:

```
<aufgabe>…woertlich…</aufgabe>
<kontext>Fundstellen, Bezug zu Projektdokumentation, falls bekannt</kontext>
<regeln>
- Projektregeln aus AGENTS.md/CLAUDE.md befolgen.
  <!-- Anpassen: eigene Kernregeln hier ergaenzen, z. B. Tests zuerst, Kommentarsprache,
  Architekturvorgaben aus docs/. -->
- Klein und fokussiert bleiben, nichts Unbeteiligtes umbauen.
- Nicht committen, nicht pushen, keine Branches anlegen.
- Betroffene Tests ausfuehren (Befehl laut AGENTS.md/CLAUDE.md) bis gruen.
</regeln>
<rueckgabe>
Kurz auf Deutsch: was geaendert wurde (Dateien), welche Tests neu sind, Testergebnis,
offene Punkte oder Annahmen. Erste Zeile: Vorschlag fuer die Commit-Nachricht
im Stil des Repos (z. B. "fix(scope): …").
</rueckgabe>
```

## 3. Codex starten

Bash mit `run_in_background: true`, dann dem Nutzer kurz Branch und Modell nennen und auf
die Benachrichtigung warten – nicht pollen:
```
codex exec -C "$S/codex-<name>" -s workspace-write \
  -m <sol-modell> -c model_reasoning_effort='"high"' \
  -o "$S/codex-<name>-result.md" - < "$S/codex-<name>-prompt.md" \
  > "$S/codex-<name>-log.md" 2>&1; echo "exit $?"
```
`-c`/`-m` gelten nur fuer diesen Aufruf; `~/.codex/config.toml` nicht aendern.
Im Sandbox-Modus kann Codex nicht ins Netz und nicht ins Haupt-`.git` schreiben – darum
committet nicht Codex, sondern dieser Skill.

## 4. Ergebnis pruefen

- `$S/codex-<name>-result.md` lesen; Kopf des Logs auf `model:` und `reasoning effort:` pruefen.
- `git -C <worktree> status --short` und `diff --stat`. Kein Diff → melden, Worktree
  entfernen, aufhoeren.
- Die Projektpruefung im Worktree laufen lassen, falls vorhanden (z. B. ein Testskript
  laut CLAUDE.md/AGENTS.md, sonst die betroffenen Tests). Rot → **kein PR**; Fehler und Codex-Rueckgabe dem Nutzer
  zeigen und anbieten, Codex mit der Fehlermeldung nachbessern zu lassen
  (gleicher Worktree, neuer `codex exec`-Lauf, Prompt = Fehlerausgabe + „bitte beheben“).
- Kurz auf Ausreisser schauen (Dateien ausserhalb des Auftrags, Secrets, grosse Binaries) –
  kein inhaltliches Review.

## 5. Commit, Push, PR

Die Skill-Ausfuehrung ist der Auftrag dazu; nur bei Ausreissern vorher fragen.
```
git -C <worktree> add -A
git -C <worktree> commit -m "<Codex-Vorschlag>" -m "Umgesetzt von Codex (<sol-modell>, effort high)." -m "Co-Authored-By: …laut aktueller Attribution…"
git -C <worktree> push -u origin <branch>
gh pr create --head <branch> --base <basis> --title "<Commit-Titel>" --body-file <datei>
```
PR-Text auf Deutsch: Aufgabe, Aenderungen, Tests, offene Punkte aus der Codex-Rueckgabe,
Hinweis „Umgesetzt von Codex (<modell>, effort high)“, Attributionszeile.
Kein Auto-Merge.

## 6. Aufraeumen und melden

- `git worktree remove "$S/codex-<name>"` (Branch bleibt fuer den PR bestehen).
- Antwort auf Deutsch, knapp: PR-Link, Branch, Modell, geaenderte Dateien, Testergebnis,
  offene Punkte von Codex. Hinweis: Review mit `/pr-review-codex <nr>` starten.
