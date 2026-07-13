# BuildSpace Skills

AI agent skills for building apps with the [BuildSpace SDK](https://docs.buildspace.studio) (`@buildspacestudio/sdk`). These skills teach AI coding assistants how to use BuildSpace's authentication, event tracking, file storage, email notification, and billing APIs in Next.js projects. The recipes point at the working examples that ship in the [BuildSpace starter](https://github.com/BuildSpaceStudio/build-space-starter), where every capability has one in-repo vertical slice.

> This repository is a read-only mirror, automatically synced from the BuildSpace monorepo. Issues and discussions are welcome here, but pull requests can't be merged directly — direct commits are overwritten by the next sync.

## Install

| Channel | Command |
|---------|---------|
| **skills.sh CLI** | `npx skills add buildspacestudio/skills` |
| **Claude Code marketplace** | `/plugin marketplace add buildspacestudio/skills` then `/plugin install buildspace-skills@buildspace-skills` |
| **Cursor** | `/add-plugin buildspacestudio/skills` or add as a Remote Rule via Settings → Rules & Commands → Project Rules → Remote Rule (GitHub) |
| **Specific skill only** | `npx skills add buildspacestudio/skills -s buildspace-sdk` |
| **Manual** | Clone this repo, copy `skills/<name>/` into your project's `.claude/skills/` |

## Skills

### [`buildspace-sdk`](skills/buildspace-sdk/SKILL.md)

SDK reference covering authentication, events, storage, notifications, and billing. Includes entry points, environment variable setup, error handling, and complete method signatures in the companion [API reference](skills/buildspace-sdk/api-reference.md).

**Use when:** Adding or modifying auth flows, tracking user events, handling file uploads, sending transactional emails, or selling subscriptions.

### [`buildspace-examples`](skills/buildspace-examples/SKILL.md)

Patterns and recipes for common features: the new-feature vertical-slice playbook, AuthProvider + useAuth hook, route protection with `proxy.ts`, server actions with next-safe-action, event tracking, file storage, email notifications, and billing (checkout, portal, entitlements).

**Use when:** Adding a feature, implementing authentication UI, route protection, server actions, billing, or any feature that combines multiple SDK services.

### [`buildspace-cli`](skills/buildspace-cli/SKILL.md)

CLI reference for `buildspace auth`, `buildspace deploy`, `buildspace env`, and `buildspace init`.

**Use when:** Deploying apps, managing environment variables, or setting up new projects from the command line.

## Documentation

Full BuildSpace documentation: [docs.buildspace.studio](https://docs.buildspace.studio)

LLM-optimized docs: `https://docs.buildspace.studio/llms-full.txt`

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add new skills.

## License

[MIT](LICENSE)
