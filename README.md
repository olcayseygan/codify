# codify

A Claude Code skill that checks code against a fixed rule set and **fixes** every violation it finds,
in place. It hands back a diff, not a review.

It is the closing half of a pair: `code-audit` opens the question ("what is wrong with this?"),
`codify` closes it ("these 23 things are either true of this code or they are now").

## What it does

1. Reads the whole target — a path, a diff, or the uncommitted working tree.
2. Loads `rules/core.md` plus one idiom file for the language in hand.
3. Checks **every rule against every file**. A rule not looked for is not silently clean.
4. Tries to kill each FAIL before acting on it; a FAIL that does not survive produces no edit and no
   line of output.
5. Fixes what survives, in dependency order — deletions before renames before spacing — then re-runs
   every rule a fix touched.
6. Runs the project's own type check, lint and tests to prove nothing broke.
7. Prints the edit list. No report, no findings table, no "shall I apply these?".

## Install

Drop the directory into your skills folder:

```
~/.claude/skills/codify/         # user-wide
.claude/skills/codify/           # or per project
```

```
codify/
  SKILL.md              procedure: targets, precedence, fix order, verification, output
  rules/
    core.md             rules 1-13 and 17-26
    idiom/
      python.md         rule 12 for Python
      typescript.md     rule 12 for JS/TS, Vue, React
      csharp.md         rule 12 for C# and Unity
```

## Use

```
/codify                        the uncommitted diff
/codify src/parser.ts          one file
/codify src/services/          a directory
/codify --only 13,25           two rules, this run only
/codify --skip 11              everything but one
```

It also answers plain requests: "check this against the rules", "is this SOLID", "any magic
numbers", "too many null checks", "is this injectable", "kurallara uyuyor mu".

## The rules

Numbering is global and stable — rules cross-reference each other by number, and so does the output.
**14-16 are retired and never reused.**

| # | Rule |
| --- | --- |
| 1 | Types — everything is typed |
| 2 | Names mean something — no abbreviations |
| 3 | No repetition |
| 4 | SOLID |
| 5 | Single entry |
| 6 | Test driven |
| 7 | Function names are verbs |
| 8 | Variable names are nouns |
| 9 | Booleans start with is / has / can |
| 10 | No magic numbers or strings |
| 11 | Blank lines separate blocks, never code |
| 12 | Idiomatic — write the language, not a translation of another one |
| 13 | No defensive guards — let it fail loudly |
| 17 | Everything opened is closed |
| 18 | No dead code |
| 19 | No hidden mutation |
| 20 | Errors carry what broke |
| 21 | Comments say why, not what |
| 22 | No flag parameters |
| 23 | No floating promises |
| 24 | Time, randomness and identity come from outside |
| 25 | Untrusted input is never interpolated |
| 26 | Dependencies point one way |

When two rules disagree the lower number wins, with five named exceptions — 13 beats 12, 25 beats 13,
17 beats 13, 18 beats 21, and rule 10 yields on hardcoded secrets. See section 3 of `SKILL.md`.

## Project overrides

Some of these rules are house style, and a team that disagrees with one will stop running the skill
entirely rather than argue with it every run. Put a `.codify.md` at the repository root:

```
disable 11   house style: blank lines separate paragraphs inside long test bodies
relax 2      idx, ok and db are established here
```

- `disable N` — not checked, produces no edits.
- `relax N` — checked, but the exceptions named in the reason are accepted.
- **A directive with no reason is not honoured.** The reason is the whole point; its absence usually
  means someone silenced a rule they lost an argument with.
- Rule 25 can be disabled, but never silently — if it is off, the run says so on its own line.

## Output

```
parser.ts:44         1   parse now returns ParseResult
service.ts:12        2   cfg -> configuration
orders.ts:14        13   swallowing catch deleted
useFeed.ts:31       17   scroll listener removed on unmount
search.py:29        25   query parameterised

tsc --noEmit + vitest: green
overrides   11 disabled, 2 relaxed (.codify.md)

NEEDS DECISION   config.ts:4       10   hardcoded API key — which env var should this read from?
OUT OF TARGET    api/client.ts:9   18   the last call site lives outside the target
```

Two kinds of finding are left unfixed, both named out loud: `NEEDS DECISION`, where the fix turns on
something only you know, and `OUT OF TARGET`, where the violation's real home is a file you did not
point at. Rules that passed produce no line — a passing rule produced no edit.

## Scope and safety

- **Up to 20 edits or 8 files** — fixed without asking. Past either, it prints the count and asks one
  question about *how much*, never about *whether*.
- **No commits by default.** The edits sit in the working tree. The one exception is a clean tree plus
  an over-budget run you chose to fix wholesale: then one commit per rule, so any can be reverted
  alone.
- **Nothing is claimed green in silence.** If there is no type check and no test suite, the run says
  so in one line, and it will not install tooling the project does not have.
- If an edit breaks the build in a way that is not obvious, that one edit is reverted, the rest stay,
  and the error comes back as a `NEEDS DECISION`.

## Cost

Roughly 50 KB per run — `SKILL.md`, `rules/core.md`, and one idiom file. Only the ~960-character
frontmatter description sits in context when the skill is not running.
