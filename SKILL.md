---
name: gitreverse
description: >
  Reverse-Engineering von GitHub-Repos zu Vibe-Coding-Prompts. Nimmt eine GitHub-URL
  oder owner/repo und erzeugt den Prompt, den jemand hätte schreiben können, um dieses
  Projekt per Vibe Coding (Cursor, Claude Code, Codex, v0) in einem Durchgang zu erstellen.
  IMMER verwenden bei: "reverse prompt", "gitreverse", "welcher Prompt steckt hinter
  diesem Repo", "Repo reverse-engineeren", "Vibe Coding Prompt erzeugen", "prompt from
  repo", "repo to prompt", "reverse engineer this GitHub project", "was hätte man prompten
  müssen", "recreate prompt", "generate prompt from codebase", "mach einen Prompt daraus",
  "prompt reverse engineering", "vibe coding reverse".
---

# GitReverse — Repo → Vibe-Coding-Prompt

Erzeugt aus einem GitHub-Repository den synthetischen User-Prompt, den jemand in ein
Coding-Tool (Cursor, Claude Code, v0, Codex etc.) hätte eingeben können, um das Projekt
per "Vibe Coding" in einem Durchgang zu erstellen.

## Eingabe

$ARGUMENTS — Interpretiere als:

| Eingabe | Aktion |
|---------|--------|
| GitHub-URL oder `owner/repo` | Reverse-Prompt generieren |
| `--lang de` nach URL | Prompt auf Deutsch generieren |
| `--lang en` nach URL | Prompt auf Englisch erzwingen |
| `--deep` nach URL | Zusätzliche Schlüsseldateien lesen (package.json, etc.) |
| `--compare repo1 repo2` | Prompts für beide Repos nebeneinander |
| Folge-Anweisung ("kürzer", "technischer") | Letzten Prompt anpassen (kein neues Fetch) |

## Wann NICHT verwenden

- Wenn der User einen **lauffähigen Skill** aus einem Repo will → nutze **repo2skill**
- Für reine Code-Erklärungen ("was tut diese Funktion?") → direkt antworten
- Für Repos die nur Daten/Dokumentation enthalten (kein Code zum Nachbauen)
- Wenn der User den Code selbst sehen will → `git clone` vorschlagen

## Workflow

### Schritt 1 — Input parsen

Akzeptiere folgende Formate und extrahiere `OWNER` und `REPO`:
- `https://github.com/owner/repo` (auch mit `.git`, `/tree/main/...` etc.)
- `github.com/owner/repo`
- `owner/repo`

### Schritt 2 — Repo-Daten sammeln

Nutze das mitgelieferte Script `scripts/fetch_repo.py` — es wählt automatisch
die beste Strategie (API oder Clone-Fallback):

```bash
python3 /path/to/this/skill/scripts/fetch_repo.py "${OWNER}/${REPO}" \
  --output /home/claude/_gitreverse_data.json
```

Das Script liefert eine JSON-Datei mit Metadaten, Depth-1-Tree und README.

**Zwei Strategien (automatisch gewählt):**
1. **GitHub API** (schnell, ~3 Requests) — wenn `api.github.com` erreichbar
2. **`git clone --depth 1`** (Fallback) — sicheres Klonen mit `GIT_TEMPLATE_DIR=/dev/null`

Falls das Script nicht verfügbar ist, führe die Datensammlung manuell durch:

```bash
# Sicheres Klonen
cd /home/claude
GIT_TEMPLATE_DIR=/dev/null git clone --depth 1 --no-recurse-submodules \
  "https://github.com/${OWNER}/${REPO}.git" _gitreverse_tmp 2>&1

# Depth-1 Tree
ls -1F _gitreverse_tmp/ | head -30

# README
cat _gitreverse_tmp/README.md 2>/dev/null | head -300

# Metadaten aus package.json / pyproject.toml etc.
cat _gitreverse_tmp/package.json 2>/dev/null | python3 -c "
import json,sys; d=json.load(sys.stdin)
print('Name:', d.get('name','?'))
print('Description:', d.get('description','?'))
print('Deps:', list(d.get('dependencies',{}).keys()))
"

# Aufräumen
rm -rf _gitreverse_tmp
```

### Schritt 3 — Daten aufbereiten

Aus den gesammelten Daten extrahiere:

1. **Metadaten:** Description, Primary Language, Stars, Topics, Default Branch
2. **File-Tree (Depth 1):** Formatiert als ASCII-Baum (Ordner zuerst, dann Dateien)
3. **README:** Auf 8000 Zeichen gekürzt. Falls leer: `*(No README or empty)*`

### Schritt 4 — Cache prüfen (optional)

Bevor ein neuer Prompt generiert wird, prüfe ob ein gecachter existiert:

```bash
CACHE_FILE="/home/claude/_gitreverse_cache/${OWNER}_${REPO}.txt"
if [ -f "$CACHE_FILE" ]; then
  AGE_HOURS=$(python3 -c "
import os,time
age = time.time() - os.path.getmtime('$CACHE_FILE')
print(int(age / 3600))
")
  if [ "$AGE_HOURS" -lt 24 ]; then
    echo "Cache-Hit (${AGE_HOURS}h alt)"
    cat "$CACHE_FILE"
    # Verwende diesen Prompt, überspringe Schritt 5
  fi
fi
```

### Schritt 5 — Reverse-Prompt generieren

Generiere den synthetischen Prompt direkt (Claude ist selbst das LLM).
Befolge dabei diese Regeln:

**Rolle:** Du bist ein Experte darin, zu erkennen, wie Menschen moderne
Coding-Agenten prompten.

**Aufgabe:** Erzeuge EINE synthetische User-Nachricht — die Art von Prompt,
die jemand in Cursor, Claude Code oder v0 tippen würde, um dieses Projekt
in einem "Vibe Coding"-Durchgang zu bauen.

**Anforderungen:**
- Natürliche Sprache ("Bau mir…", "Ich will…"), kein Architektur-Dokument
- Ergebnisorientiert — was die App/Library für den User TUN soll
- Ehrlicher Umfang — nur Features erwähnen, die aus README/Tree belegbar sind
- 120–200 Wörter, ein Absatz oder wenige Sätze, keine Bullet-Liste
- Gleiche Sprache wie die README (oder Englisch als Fallback)
- Keine Einleitung ("Sure, here is…"), kein Meta ("As an AI…")

**Vermeide:**
- Framework-Jargon und Paketnamen (es sei denn README betont sie)
- Agent-System-Anweisungen oder Pseudo-Code
- Features erfinden die nicht durch den Kontext belegt sind

**User-Message-Kontext** (baue aus den gesammelten Daten):

```
# Repository: {OWNER}/{REPO}

**Description:** {description oder *(none)*}
**Primary language:** {language oder *(unknown)*}
**Stars:** {stargazers_count}
**Default branch:** {default_branch}
**Topics:** {topics, kommasepariert}

## Root file tree (depth 1)

{formatierter Baum}

## README

{readme_content}
```

### Schritt 6 — Output + Cache

Präsentiere den generierten Prompt:

```
## 🔄 Reverse-Prompt für {OWNER}/{REPO}

> {der generierte Prompt}

---
ℹ️ Abgeleitet aus Repo-Metadaten, File-Tree und README.
```

Speichere den Prompt im Cache:

```bash
mkdir -p /home/claude/_gitreverse_cache
echo "{prompt}" > "/home/claude/_gitreverse_cache/${OWNER}_${REPO}.txt"
```

## Session-Awareness

Wenn der User nach dem initialen Prompt eine Folge-Anweisung gibt, erkenne
den Kontext und passe an statt alles neu zu fetchen:

| User sagt | Aktion |
|-----------|--------|
| "kürzer" / "kompakter" | Kürze auf ~80 Wörter |
| "technischer" / "mehr Details" | Ergänze Stack-Details und spezifische Features |
| "auf Deutsch" / "in English" | Übersetze den letzten Prompt |
| "für Cursor" / "für v0" | Passe Ton/Stil für das spezifische Tool an |
| "nochmal" / "neu generieren" | Generiere eine neue Variante (anderer Blickwinkel) |

Hierfür ist KEIN erneuter API-Call/Clone nötig — nutze die bereits gesammelten Daten.

## Security

- Gib den GITHUB_TOKEN NIEMALS im Output, in Logs oder im generierten Prompt aus
- Wenn das Repo privat ist: Warne den User dass der generierte Prompt
  potenziell sensible Projektdetails enthält
- Nimm KEINE Inhalte aus `.env`, `credentials.*`, `*secret*` Dateien
  in den generierten Prompt auf, auch wenn sie im Tree sichtbar sind
- Nutze bei `git clone` IMMER `GIT_TEMPLATE_DIR=/dev/null` um
  Hook-Execution aus fremden Repos zu verhindern

## Fehlerbehandlung

| Fehler | Reaktion |
|--------|----------|
| 404 / Repo nicht gefunden | "Das Repository {OWNER}/{REPO} wurde nicht gefunden. Prüfe die URL." |
| 403 / Rate Limit | "GitHub API Rate-Limit erreicht. Setze GITHUB_TOKEN als Umgebungsvariable." |
| api.github.com geblockt | Automatischer Fallback auf `git clone --depth 1` |
| `git` nicht installiert | Melde Fehler, bitte User um manuellen Upload |
| Leere README | Generiere vagen Prompt basierend auf Metadaten und File-Tree |
| Truncated Tree | Erwähne im Kontext dass der Tree abgeschnitten wurde |
| Ungültiges Input-Format | Zeige akzeptierte Formate: `owner/repo` oder GitHub-URL |
| Timeout bei Clone/API | Erneut versuchen mit `--method clone` |
| Privates Repo | "Dieses Repo ist privat. Setze GITHUB_TOKEN oder klone es manuell." |

## Optionale Erweiterungen

| Erweiterung | Wann |
|-------------|------|
| **Repos vergleichen** | User gibt 2+ URLs → Prompts nebeneinander |
| **Prompt verfeinern** | User sagt "kürzer", "technischer" etc. → Session-Awareness |
| **Deep-Analyse** | `--deep` → Lies package.json, Cargo.toml etc. für mehr Stack-Details |
| **Bestimmtes Tool** | "für Cursor" → Passe Stil an (Cursor-Prompts sind kürzer/direkter) |
