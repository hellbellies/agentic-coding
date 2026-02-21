# Agentic Coding

## Skills

### Install with npx skills

Install all my skills:

```bash
npx skills add hellbellies/agentic-coding
```

Install all specific skill, e.g. `prd`

```bash
npx skills add hellbellies/agentic-coding --skill prd
```

> [!info] Skills installed with `npx skills` are added to the catalogue at [skills.sh](https://skills.sh)

## Crush Commands

Crush commands are simple instructions to the [Crush CLI](https://github.com/charmbracelet/crush) that can be triggered doing:

- `ctrl + p` to open the command palette
- `tab` to navigate to the User commands

### Installing Commands

Copy command .md files into `~/.crush/commands`

### Writing Commands

Give instructions to Crush using natural language. You can use `$<VARIABLE>` to have Crush ask you for variables. E.g.

```markdown
$FEATURE. Use the prd skill to create a detailed product requirements document.
```
