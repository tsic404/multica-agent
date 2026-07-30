# multica-agent

Multica agent, skill, and squad prompts — version-controlled prompt store.

## Structure

```
multica-agent/
├── agent/          # Agent instructions (prompts)
│   ├── lynx.md
│   ├── aureus.md
│   ├── vulcan.md
│   ├── vexel.md
│   ├── radian.md
│   ├── verity.md
│   ├── dockerfile.md
│   └── coordinator.md
├── skill/          # Skill content
│   ├── backend-dev-standards.md
│   ├── code-review-checklist.md
│   ├── dockerfile-adaptation.md
│   ├── docker-upstream-dockerfile-sync.md
│   ├── econ-abm-qa-testing.md
│   ├── frontend-dev-standards.md
│   ├── qa-testing-workflow.md
│   ├── rffmpeg-qa-testing.md
│   ├── task-splitting-guide.md
│   └── torrentfs-qa-testing.md
└── squad/          # Squad instructions
    └── dev-team.md
```

## Workflow

This repo is the source of truth for Multica prompts. Design docs live at [multica-cooperation](https://github.com/tsic404/multica-cooperation).

To deploy:
```bash
multica agent update <uuid> --instructions "$(cat agent/<name>.md)"
multica skill update <uuid> --content "$(cat skill/<name>.md)"
multica squad update <uuid> --instructions "$(cat squad/<name>.md)"
```
