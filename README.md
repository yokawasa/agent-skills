# agent-skills

This is a repository of [Agent Skills](https://agentskills.io) that I've developed. Note that this repository is unofficial and still in beta.

## Skills

The following table lists the skills currently available in this repository. More skills will be added over time.

| Skill Name | Description |
|------------|----------------------------------------------------------------------------------|
| [digital-agency-api-guide](./skills/digital-agency-api-guide) | A skill for supporting Web API design, review, and specification generation based on the Digital Agency’s “[API Technical Guidebook (デジタル庁「APIテクニカルガイドブック」)](https://www.digital.go.jp/assets/contents/node/basic_page/field_ref_resources/fe5f0631-c978-42db-8416-6759cfa7e53a/f6ca1b7b/20241001_policies_development_management_outline_04.pdf)” published on Sept 30, 2024.|

## How to Install/Update Skills

### Install

```bash
npx skills add yokawasa/agent-skills --skill <skill-name>

# Or install for a specific agent (Claude Code, Codex, etc.)
npx skills add yokawasa/agent-skills --skill <skill-name> -a claude-code
npx skills add yokawasa/agent-skills --skill <skill-name> -a codex
```

For example, to install the `digital-agency-api-guide` skill:

```bash
npx skills add yokawasa/agent-skills --skill digital-agency-api-guide
```

### List Skills

List available skills in this repository (without installing):

```bash
npx skills add yokawasa/agent-skills --list
```

List skills already installed on your machine:

```bash
npx skills list
```

### Update / Delete

```bash
# Update
npx skills update <skill-name>
# Delete
npx skills remove <skill-name>
```

## Contributing

If you find any mistakes or areas for improvement, feel free to submit a pull request! If you have any questions or suggestions, please open an issue.

## License

[MIT](./LICENSE)

