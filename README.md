# claude-workflow

A Claude Code plugin with a workflow for issue/task work in the
`llm-chat-python-tutorial-docker` project. Currently ships one skill:

- **`work-on-issue`** — multi-agent pipeline: Opus plans, Sonnet
  implements, then two parallel Sonnet reviews (acceptance compliance +
  code review) with a feedback loop. Runs in three modes: no argument
  (pick from a list of open issues), one or more issue numbers, or a
  free-text task description.

## Structure

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

## Installation

The plugin is installed via Claude Code's plugin system. First add this
repository as a marketplace, then install the plugin from it.

### 1. Add the marketplace from GitHub

In Claude Code, run:

```
/plugin marketplace add michalstaniecko/claude-workflow
```

You can also pass a full HTTPS or Git URL:

```
/plugin marketplace add https://github.com/michalstaniecko/claude-workflow.git
```

### 2. Install the plugin

```
/plugin install claude-workflow
```

Or open the interactive menu:

```
/plugin
```

and pick `claude-workflow` from the list.

### 3. Verification

After installation the `work-on-issue` skill should appear in the list
of available skills. You can invoke it with:

```
/work-on-issue 12
```

or without an argument, to pick an issue from the list:

```
/work-on-issue
```

or with a free-text task description (no issue):

```
/work-on-issue fix slug validation in the tenant API
```

## Updates

```
/plugin marketplace update michalstaniecko/claude-workflow
/plugin update claude-workflow
```

## Uninstall

```
/plugin uninstall claude-workflow
```

## Requirements

The `work-on-issue` skill assumes you're working in the
`llm-chat-python-tutorial-docker` repo and have available:

- `gh` CLI (GitHub) logged in with access to the repo,
- `git`,
- `uv` (Python tooling),
- `docker compose` (optional, for running the environment),
- `npm` if you're touching `admin-ui/`.

For the full list of conventions and guardrails see
[`plugins/claude-workflow/skills/work-on-issue/SKILL.md`](plugins/claude-workflow/skills/work-on-issue/SKILL.md).
