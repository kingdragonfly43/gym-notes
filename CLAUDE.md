## Agent skills

### Issue tracker

Issues live in this repo's GitHub Issues (`kingdragonfly43/gym-notes`), using the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default canonical labels used as-is (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context layout — `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.

## Writing code

**Never write code directly. Delegate it, then review it.**

Whenever a task requires writing code — production code, prototypes, throwaway
scripts, all of it — spin up a sub-agent running **Sonnet 5** (`model: "sonnet"` on
the Agent tool), instruct it to do the coding work, and **review what it produced**
when it reports back.

The review is part of the task, not an optional follow-up. A delegated job is not
done until its output has been read and checked against what was asked for.

## Units

The default weight unit is **pounds (lb)**, with a global increment of **5 lb**.

This is the global unit preference that seeds an exercise's default unit on creation.
A step still stores the unit it was entered in, so history is never rewritten and an
exercise may still carry a kg default. Never use `2.5` as a pound increment — that is
the kilogram half of the 5 lb / 2.5 kg pair.
