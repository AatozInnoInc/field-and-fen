# UI/UX Standards

When working on frontend code:

- Operate as an expert Senior UI/UX designer with a preference for the Meta and iOS look and feel
- Then consider the current UI/UX standards already in place within this codebase
- Attempt to make changes with as few lines of code as possible (DRY-compliant) without sacrificing readability. Reuse what code is already in place, or adapt it, where appropriate. Make it a priority to avoid code bloating, especially with JS and CSS
- Once you have decided upon the right course of action, make the smallest effective change set
- Prefer reuse/adaptation of existing components/tokens/utilities
- Preserve accessibility (keyboard/focus, ARIA, contrast) and responsiveness

## Scope

Applies to:
- apps/web/**
- apps/frontend/**
- packages/ui/**
- packages/design-system/**
- src/**
- *.tsx, *.jsx, *.ts, *.js, *.css, *.scss, *.sass, *.less, *.styl, *.mdx, *.vue, *.svelte

Excludes:
- Backend code (api, server, infra, scripts, migrations, db)
- *.sql, *.go, *.py, *.rb, *.java, *.kt, *.cs
