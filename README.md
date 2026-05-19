# claude-workflow

Plugin Claude Code zawierający workflow do pracy nad issue/zadaniami w projekcie
`llm-chat-python-tutorial-docker`. Aktualnie udostępnia jeden skill:

- **`work-on-issue`** — pipeline multi-agentowy: Opus planuje, Sonnet
  implementuje, na końcu dwa równoległe review na Sonnet (zgodność z
  założeniami + code review), z pętlą zwrotną. Działa w trzech trybach: bez
  argumentu (wybór z listy otwartych issue), z numerem/numerami issue, albo
  wolnym opisem zadania.

## Struktura

```
claude-workflow/                      # marketplace root
├── .claude-plugin/
│   └── marketplace.json              # marketplace catalog
├── plugins/
│   └── claude-workflow/              # plugin
│       ├── .claude-plugin/
│       │   └── plugin.json           # plugin manifest
│       └── skills/
│           └── work-on-issue/
│               └── SKILL.md
└── README.md
```

## Instalacja

Plugin instaluje się przez system pluginów Claude Code. Najpierw dodaj
to repozytorium jako marketplace, a potem zainstaluj z niego plugin.

### 1. Dodaj marketplace z GitHuba

W Claude Code uruchom:

```
/plugin marketplace add michalstaniecko/claude-workflow
```

Możesz też podać pełny URL HTTPS lub Git:

```
/plugin marketplace add https://github.com/michalstaniecko/claude-workflow.git
```

### 2. Zainstaluj plugin

```
/plugin install claude-workflow
```

albo otwórz interaktywne menu:

```
/plugin
```

i wybierz plugin `claude-workflow` z listy.

### 3. Weryfikacja

Po instalacji skill `work-on-issue` powinien być widoczny na liście dostępnych
skilli. Możesz go wywołać przez:

```
/work-on-issue 12
```

lub bez argumentu, żeby wybrać issue z listy:

```
/work-on-issue
```

albo opisem zadania bez issue:

```
/work-on-issue popraw walidację slug w tenant API
```

## Aktualizacje

```
/plugin marketplace update michalstaniecko/claude-workflow
/plugin update claude-workflow
```

## Odinstalowanie

```
/plugin uninstall claude-workflow
```

## Wymagania

Skill `work-on-issue` zakłada, że pracujesz w repo `llm-chat-python-tutorial-docker`
i masz dostępne:

- `gh` CLI (GitHub) zalogowane na konto z dostępem do repo,
- `git`,
- `uv` (Python tooling),
- `docker compose` (opcjonalnie, do uruchamiania środowiska),
- `npm` jeśli ruszasz `admin-ui/`.

Pełna lista konwencji i guardraili — w
[`skills/work-on-issue/SKILL.md`](skills/work-on-issue/SKILL.md).
