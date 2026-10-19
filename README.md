# Arena: 16 competitors

**A fork maintained by Salman AlFares, based on [Jake Schincariol's Arena](https://github.com/Jakeschincariol/arena-skill).** This version defaults to 16 competitors. The original tournament logic and MIT license are preserved.

When Claude keeps giving you bad answers, make 16 versions of it fight to the death. One Claude
Code skill. Free, MIT, no signup, no API key, nothing to connect.

You use it when you are not happy with what Claude gave you. Instead of asking again and again,
`/arena` spins up 16 sub-agents and gives every one of them the exact same task, word for word.
Each one gets a different strategy card: a way of reasoning, a workflow and a strategy, so each one
attacks the task differently. Then they pair off. Each attacks the other's solution, each defends
and revises its own, and a judge scores the match on a written rubric. The loser is out. 16 becomes 8, then 4, 2, 1.

What comes back is the one solution that survived, the attacks it beat on the way, and how many
rounds it took.

**It never touches your files.** Every sub-agent writes inside `.arena/`. The winning answer comes
back to you, and using it is your call.

## Install

Paste this into Claude:

```
https://github.com/0alfares082/arena-skill-16

Install this skill, then confirm /arena works.
```

Or copy the folder yourself, in Claude Code:

```bash
git clone https://github.com/0alfares082/arena-skill-16.git
mkdir -p ~/.claude/skills
cp -r arena-skill-16/skills/arena ~/.claude/skills/
```

Or as a plugin:

```
/plugin marketplace add 0alfares082/arena-skill-16
/plugin install arena-skill-16@arena-skill-16
```

Claude Code namespaces plugin skills, so installed as a plugin it shows up as `/arena-skill-16:arena`.
Copy the folder instead if you want plain `/arena`. Project-local instead of global: copy the same
folder into your repo's `.claude/skills/`.

It needs Claude Code, because it spawns sub-agents with the Agent tool, and Python 3.8 or newer.
Nothing to pip install.

## Use it

```
/arena
/arena --quick write the headline for our pricing page
/arena --agents 32 fix the flaky test in tests/test_api.py
/arena --seed 7 plan my launch week, I have 6 hours a day
```

On its own, `/arena` takes your last request as the task and the answer you did not like as the one
to beat. Claude can also reach for it without the slash, when you say something like "that's a bad
answer, make them compete". When it starts itself like that, it asks before it spends anything.

| flag | what it does |
| --- | --- |
| `--agents N` | N competitors. Default 16. |
| `--quick` | 16 competitors. Compatibility alias for the default. |
| `--seed S` | Same seed, same cards and same bracket. Default random, and recorded. |
| `--wave W` | Sub-agents per wave. Default 10. Only raise it if you raised Claude Code's limit. |

## How it works

1. **Spawn.** N sub-agents, one Agent call each. Every one gets the same task text, byte for byte
   (a test checks it), plus one strategy card. There are 15 reasoning modes (first principles,
   inversion, analogy, adversarial, constraint first, worked example, Socratic, contrarian, systems
   thinking and more), 12 workflows (draft, critique, rewrite; outline first; test first; research,
   then synthesise; three drafts, pick one...) and 12 strategies (simplest thing that works, maximal
   rigour, user empathy first, edge cases first, speed...). That is 2,160 different cards. The
   dealer gives each agent its own, with no repeats.
2. **Attack.** Solutions are paired, and the pairing avoids putting two agents with the same
   reasoning mode against each other. Each side attacks the other's solution through its own card:
   what is wrong, what requirement it missed, the exact input that breaks it. Up to 7 attacks,
   labelled FATAL, MAJOR or MINOR.
3. **Defend.** Each side answers every attack it took, conceding or rebutting with evidence, then
   rewrites its solution to fix everything it conceded.
4. **Judge.** A separate judge sub-agent reads both revised solutions, checks every attack itself,
   and scores both on [the rubric](skills/arena/rubric.md): correctness 30, completeness against the
   task 25, robustness to the attacks raised 20, specificity 15, clarity 10. The judge never sees
   the cards. `bracket.py` does the arithmetic: the higher total goes through, and a solution with a
   verified fatal flaw cannot beat one without. The loser is out.
5. **Repeat.** The survivors carry their revised solutions into the next round. An odd number means
   one bye, never to the same agent twice while anyone else is waiting for one.
6. **Result.** One solution left. You get it, the attacks it survived, its card and the round count.
   If you started from an answer you rejected, a last judge compares the winner with it blind and
   the skill tells you the score, even when the old answer wins.

The default bracket, from `python3 skills/arena/bracket.py plan`:

```text
16 agents, 4 rounds, waves of 10

  round  alive  matches  bye  sub-agent calls  waves
  spawn     16        -    -               16      2
      1     16        8    -               40      5
      2      8        4    -               20      3
      3      4        2    -               10      3
      4      2        1    -                5      3
  total                                  91     16

  alive per round: 16 -> 8 -> 4 -> 2 -> 1
  plus 1 call for the final check when there is a rejected answer to beat
```

The whole tournament lives in one JSON file, managed by `bracket.py`. The main Claude session
only runs the loop and never reads the hundreds of solution files: every sub-agent writes its work
to disk and replies with one line, so the main context holds receipts and bookkeeping, not the
work. If the conversation gets compacted halfway through, `bracket.py next` picks it up from the
file.

## What it costs

The skill is free. The tokens are yours, and every tournament uses tokens.

| agents | rounds | sub-agent calls | waves of 10 |
| --- | --- | --- | --- |
| 64 | 6 | 379 | 49 |
| 32 | 5 | 187 | 28 |
| 16, the default | 4 | 91 | 16 |
| 8 | 3 | 43 | 10 |

Add one call when there is a rejected answer to beat. Every call reads the task and one or two
solutions, so the bill grows with the size of the task. The default uses 16 competitors. An explicit `--agents N` changes the count. The skill prints these numbers
before it starts, and `python3 skills/arena/bracket.py plan --agents N` prints them any time.

## The tool

`bracket.py` is standard-library Python. It is the reason the orchestrator never loses track.

```bash
python3 bracket.py plan        # rounds, calls, waves. Writes nothing
python3 bracket.py init --seed 7 --task-file task.md [--baseline-file old.md]
python3 bracket.py next                     # what to do now, with the exact command
python3 bracket.py prompts attack           # write every sub-agent brief, list the jobs in waves
python3 bracket.py pairings                 # this round's matches, including the bye
python3 bracket.py collect                  # record the judges' verdicts
python3 bracket.py record r3-m07 a042 --reason "..."   # or record one by hand
python3 bracket.py advance                  # eliminate the losers, pair the survivors
python3 bracket.py status                   # alive and eliminated, per round
python3 bracket.py winner                   # the survivor, the attacks it survived, the rounds
```

The tests run a full larger tournament through that command line with random winners and check
it ends with exactly one survivor, plus 16, 7 and 1 agents, the dealer's guarantees, and every
phase from spawn to the final check with stand-in sub-agents:

```bash
python3 -m unittest discover -s tests -v
```

## The fine print

**"16 versions of Claude" means 16 sub-agents of the model you are running.** Not 16 different
models. What makes them different is the card. The dealer guarantees that no two agents get the
same card, that every reasoning mode, workflow and strategy is dealt as evenly as possible, and that
up to 144 agents, no two agents even share two of their three parts.

**A competitor is its card plus its solution file.** Sub-agents do not remember anything between
calls, so when competitor a007 attacks in round 3, that is a fresh sub-agent handed a007's card and
a007's latest solution. 16 competitors, 91 calls.

**"The best answer" means the one that survived every match.** The judges are Claude too, scoring
against a written rubric that is in this repo for you to read and change. What you get is the
strongest answer this tournament found, not a proof that it is right. That is why it shows you the
attacks it survived, and why it tells you straight when the answer you rejected scored higher.

**The 16 competitors do not all run at the same moment.** Claude Code runs at most 10 tool calls at once by
default (the `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` setting), so the arena runs in waves of 10. The
spawn is two waves, the whole default run is 16, and it takes a while. If you raise that limit,
pass `--wave` to match.

**Expect permission prompts unless you allow edits.** Every sub-agent writes a file into `.arena/`,
and in the default permission mode that is one approval per file. Run it in accept-edits mode
(Shift+Tab).

**Sub-agents cannot see your chat.** The skill writes a standalone task file, and that file is all
16 competitors ever know. If a requirement never made it into the file, all 16 miss it. It is at
`.arena/<run>/task.md` if you want to check.

**It never edits your project.** Code changes come back as a diff or as full files inside the
winning answer. Applying them is your call, and the skill asks.

**The same seed gives the same cards and the same bracket.** Not the same answers. The model is not
deterministic.

**It does not fix a bad task.** Vague task in, 16 flavours of vague out.

## Files

```
skills/arena/SKILL.md          the orchestration steps and the exact brief every sub-agent gets
skills/arena/bracket.py        the tournament state machine, standard library only
skills/arena/strategies.json   15 reasoning modes, 12 workflows, 12 strategies. Edit freely
skills/arena/rubric.md         the five criteria every judge scores on
tests/test_bracket.py          the tests
```

## Credit

Made by Jake Schincariol, [opusjake.ai](https://opusjake.ai).

## License

MIT. Take it, change it, ship it.
