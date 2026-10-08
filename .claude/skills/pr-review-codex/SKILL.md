---
name: pr-review-codex
description: Laesst einen offenen GitHub-PR (Standard: den aeltesten) von Codex reviewen – neuestes Sol-Modell, Reasoning-Effort high – und gibt die Findings mit eigener Einschaetzung zurueck. Aufruf mit optionaler PR-Nummer, z. B. /pr-review-codex 25.
argument-hint: "[PR-Nummer]"
allowed-tools: Bash, Read
---

# PR-Review mit Codex

Argument: `$ARGUMENTS` – leer = aeltester offener PR, sonst diese PR-Nummer.

Kein Subagent: Das Review ist ein einzelner Hintergrund-Befehl, Warten kostet nichts,
und die Ausgabe bleibt unverfaelscht. Nichts am PR aendern, nichts kommentieren oder pushen.

## Ablauf

`S` = Scratchpad-Verzeichnis dieser Session (nicht `/tmp`).

1. **PR bestimmen**
   - ohne Argument: `gh pr list --state open --json number,headRefName,baseRefName,createdAt --jq 'sort_by(.createdAt)[0] | "\(.number) \(.headRefName) \(.baseRefName)"'`
   - mit Argument: `gh pr view <nr> --json number,headRefName,baseRefName --jq '"\(.number) \(.headRefName) \(.baseRefName)"'`
   - Kein offener PR → dem Nutzer sagen und aufhoeren.

2. **Neuestes Sol-Modell** zur Laufzeit ermitteln (nicht hart verdrahten):
   `codex debug models 2>/dev/null | grep -o '"slug":"[^"]*-sol"' | cut -d'"' -f4 | sort -V | tail -1`
   Leer → abbrechen und melden (Codex-CLI fehlt oder nicht angemeldet; `/codex:setup`).

3. **Eigener Worktree** – der Arbeitsbaum des Nutzers bleibt unberuehrt:
   ```
   git fetch origin --quiet
   git worktree add --detach "$S/pr-<nr>" origin/<headRef>
   git -C "$S/pr-<nr>" diff --shortstat origin/<baseRef>...HEAD
   ```

4. **Review im Hintergrund** starten (Bash mit `run_in_background: true`), dann dem
   Nutzer kurz PR, Modell und Umfang nennen und auf die Benachrichtigung warten – nicht pollen:
   ```
   cd "$S/pr-<nr>" && codex review --base origin/<baseRef> \
     -c model='"<sol-modell>"' -c model_reasoning_effort='"high"' \
     > "$S/pr-<nr>-review.md" 2>&1; echo "exit $?"
   ```
   Das Plugin-Skript `codex-companion.mjs review` kennt kein `--effort`, darum direkt die CLI.
   `-c` gilt nur fuer diesen Aufruf; `~/.codex/config.toml` nicht aendern.

5. **Ergebnis lesen**: Kopf (Zeilen mit `model:` und `reasoning effort:` pruefen) und den
   Schlussblock ab `Full review comments:` – die Datei ist lang, nicht komplett einlesen.

6. **Findings pruefen**: Jedes Finding an der genannten Stelle im Worktree nachsehen und als
   *bestaetigt*, *plausibel* oder *zweifelhaft* einstufen (kurze Begruendung).

7. **Aufraeumen**: `git worktree remove --force "$S/pr-<nr>"`. Codex legt bei
   Nachstellungen eigene Dateien an (z. B. unter `/tmp` bzw. `/private/tmp`) – auflisten,
   aber nur auf Wunsch loeschen.

## Antwort an den Nutzer

Auf Deutsch, knapp:
- Kopfzeile: PR (als Link `https://github.com/<owner>/<repo>/pull/<nr>`), Modell, Effort, Umfang.
- Je Finding: Prioritaet, Titel, Datei:Zeile als Markdown-Link relativ zum Repo
  (Worktree-Pfad abschneiden), Codex' Kernaussage, eigene Einstufung.
- Codex' Gesamtfazit (z. B. Testlage) in einem Satz.
- Pfad zur vollen Ausgabe, Hinweis auf Codex-Tempdateien.
- Abschluss: anbieten, die Findings auf dem PR-Branch zu beheben – nicht ungefragt anfangen.
