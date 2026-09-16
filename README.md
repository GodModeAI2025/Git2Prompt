# Git2Prompt (GitReverse)

Git2Prompt (intern als `gitreverse`) erzeugt aus einem GitHub-Repository einen synthetischen Vibe-Coding-Prompt.  
Die Idee: Aus Struktur, Metadaten und README eines Repos wird rekonstruiert, welchen Prompt man in Tools wie Cursor, Codex, Claude Code oder v0 hätte verwenden können.

## Features

- Akzeptiert GitHub-URL oder `owner/repo`
- Generiert einen plausiblen "Reverse Prompt" in natürlicher Sprache
- Optional sprachgesteuerte Ausgabe (`--lang de` oder `--lang en`)
- Optional tieferes Einlesen wichtiger Projektdateien (`--deep`)
- Vergleichsmodus für zwei Repositories (`--compare repo1 repo2`)
- Session-aware Anpassung eines zuletzt erzeugten Prompts (z. B. "kürzer", "technischer")

## Eingabeformate

Unterstützt werden:

- `https://github.com/owner/repo`
- `github.com/owner/repo`
- `owner/repo`

## Verwendung

### Standard

```bash
gitreverse https://github.com/owner/repo
```

### Sprache erzwingen

```bash
gitreverse https://github.com/owner/repo --lang de
gitreverse https://github.com/owner/repo --lang en
```

### Deep-Analyse

```bash
gitreverse https://github.com/owner/repo --deep
```

### Zwei Repositories vergleichen

```bash
gitreverse --compare owner/repo1 owner/repo2
```

## Ausgabe

Die Ausgabe ist ein einzelner, synthetischer User-Prompt:

- natürlich formuliert
- feature-orientiert statt implementierungsgetrieben
- auf belegbare Repo-Fakten beschränkt
- typischerweise 120-200 Wörter

## Sicherheit

- Keine Ausgabe von `GITHUB_TOKEN`
- Keine Übernahme sensitiver Inhalte aus `.env`, `credentials.*` oder `*secret*`
- Bei Clone-Fallback: sichere Git-Template-Einstellungen (`GIT_TEMPLATE_DIR=/dev/null`)
- Clone mit `core.symlinks=false`; Symlinks werden nie gelesen (kein Abfluss lokaler Dateien über präparierte Links)
- README und Dateinamen fremder Repos gelten als Daten: eingebettete Anweisungen werden ignoriert, keine Befehle aus dem Repo ausgeführt
- Quellenangabe mit Commit-SHA des analysierten Stands

## Fehlerbehandlung

Typische Fälle:

- `404`: Repository nicht gefunden
- `403`: GitHub-Rate-Limit erreicht
- API nicht erreichbar: Fallback auf `git clone --depth 1`
- Leere README: Prompt basiert stärker auf Metadaten und Dateibaum

## Projektstruktur

Aktueller Stand dieses Repositories:

```text
.
├── LICENSE
├── README.md
└── SKILL.md
```

## Lizenz

Dieses Repository steht unter der MIT-Lizenz. Siehe [LICENSE](LICENSE).
