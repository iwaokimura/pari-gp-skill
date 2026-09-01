# pari-gp-skill

A [Claude Code skill](https://support.claude.com/en/articles/12512176-what-are-skills)
of recurring PARI/GP scripting pitfalls: `.gp` script-file parsing rules that
don't apply in the interactive REPL, a stack-size setting that silently kills a
whole run when placed inside a function, semantic traps (`subst` vs `substvec`,
`ffgen`, `my()`/closures, variable priority in `nffactor`), a `vecsum` type
trap, `~` being transpose and not bitwise NOT, a place where exactness silently
leaks into floating point, and errors of every kind -- syntax, arity, runtime --
making gp skip the whole file and still exit 0 (with `gp2c` as the lint that
catches the static half). Also documents the Claude Code convention
of creating `.gp` files with the Write tool rather than shell heredoc.

Every item here was hit more than once, in independent sessions, while
authoring PARI/GP verification scripts for a number-theory research project —
this is a distilled "don't step here again" checklist, not a general PARI/GP
tutorial.

The skill itself lives in [`pari-gp/SKILL.md`](pari-gp/SKILL.md).

## Install

### Manual

```sh
git clone https://github.com/iwaokimura/pari-gp-skill.git
cp -r pari-gp-skill/pari-gp ~/.claude/skills/
```

### As a Claude Code plugin

In Claude Code:

```
/plugin marketplace add iwaokimura/pari-gp-skill
/plugin install pari-gp@pari-gp-skill
```

## License

MIT — see [LICENSE](LICENSE).
