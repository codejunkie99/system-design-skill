# system-design

A skill that makes a coding agent design the system with you before it writes code. It runs a short interview in 8 stages, does the capacity math, and records each trade-off. Then it writes a `DESIGN.md` for you to approve, and the coding session builds from that file.

The skill is one `SKILL.md` file with `name` and `description` frontmatter. It is written for Claude Code.

## What the skill does

- It asks 2 or 3 questions at a time. Each question has a suggested answer, so you can reply "default" and continue.
- It turns answers such as "10k daily users" into requests per second, peak load, storage per year, and bandwidth. Later choices refer to these numbers.
- It sets a tier, from toy to large, after the first stage. Then it marks each stage as needed, light, or skip. A weekend app does not get Kafka or sharding.
- It writes a decision record for each real choice: the options, the choice, the reason, what you give up, and when to revisit it.
- It draws a Mermaid diagram from the decisions.
- It runs a pre-mortem that covers the single point of failure, 10x traffic, bad input, duplicate retries, data loss, and cost.
- It writes no code until you approve `DESIGN.md`. The file has the requirements, numbers, diagram, decisions, risks, build order, and test list.

## Install

To use the skill in all your projects, clone it into your personal skills folder:

```sh
git clone https://github.com/codejunkie99/system-design-skill.git ~/.claude/skills/system-design
```

To use it in one project only, clone it into `.claude/skills/system-design` in that project.

## Use

Tell the agent what you want to build. The skill starts when you ask for a design, an architecture, "how should I build", or "plan this system". To start it directly, type `/system-design`.

## The 8 stages

1. Define the problem
2. Estimate the scale
3. Map the system
4. Store and retrieve
5. Handle traffic
6. Coordinate work
7. Make it reliable
8. Operate and protect

## Known limits

- The skill has had one test run, a simulated interview for a URL shortener. Nobody has used it in a real session yet, so the questions may need changes.
- Some values are suggested targets, not researched numbers: the ~300 ms response time, the rate limits, and the cut-offs between tiers.
- Peak traffic defaults to 2x the average. No source gave a standard ratio.
- The skill does not say how many requests one server can handle. It tells you to run a load test.
- The latency table uses numbers from about 2012 hardware. The skill says so.

## License

MIT. See [LICENSE](LICENSE).
