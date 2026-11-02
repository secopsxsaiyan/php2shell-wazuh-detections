# Contributing

Tuning reports are the most useful thing you can send. A rule that fires on
your estate and not on the test host is information nobody here can generate.

## Before opening a pull request

CI runs on every pull request: `xmllint`, a structural pass over the rules,
`sigma check`, an ASCII-only check, and a check that every rule id is documented
in the README. You can run the Sigma half locally:

```sh
pip install sigma-cli
sigma check sigma/*.yaml
```

For the rules, validate on a manager rather than by eye. `wazuh-logtest` is the
authority: paste the log line, confirm the rule you expect fires, and confirm it
does not fire on a line it should ignore. Include both results in the pull
request.

## What a rule in this pack has to do

1. **Key on the action, not a side effect.** An HTTP status, an out-of-memory
   line, a crash, or a tool's own success banner is evidence that something
   happened, not a detection of who did it. Where both exist, the rule keys on
   the action and the side effect is documented as context.
2. **Narrow beyond its parent.** A child of a broad `if_sid` with no condition
   of its own raises the level on everything that parent decodes. There is a
   test for this, because it is an easy mistake that looks fine in review.
3. **Come with a false-positive number, not an adjective.** "Low noise" is not
   reviewable. "40 alerts in four minutes on an idle host, all uid 0" is. If
   you cannot measure it, say so in the pull request.
4. **Come with the events that prove it.** See the evidence section below.
5. **State its limits in the README.** What it misses and what a legitimate
   match looks like belong next to the rule, not in a commit message.

A rule that fires during ordinary operation is not accepted, even with a good
story attached. If you need one for triage, keep it local.

## Evidence to include

For every rule change, show two things: an event the rule catches, and an event
it must ignore that someone would plausibly expect it to catch. The second is
the one that matters. A shell-spawn rule looks correct until you run it on an
idle host and find out that cron, systemd and the agent's own wodles execute
shells constantly.

Paste both as sanitized log lines or alert JSON, with the `wazuh-logtest`
verdict for each. Replace real hostnames, internal addresses, usernames and
paths. Addresses from `192.0.2.0/24`, `198.51.100.0/24` and `203.0.113.0/24` are
reserved for documentation and are safe to use.

## Rule ids

Rules here use 102005 and 102007 because that range was free on the manager
they were written against. Ids are not a standard, and you should change them
if they collide with your own. If you add a rule, pick the next free id in this
pack and say in the pull request that you checked it is free in a stock Wazuh
install.

## Style

Plain ASCII. No em dashes. Short lines in Markdown. Comment headers in rule
files explain the signal, the requirement and the tuning, in that order,
because that is the order someone deploying the rule needs them.

## Reporting a false positive

Open an issue using the false-positive template and include the alert JSON with
your identifiers removed, plus what the process legitimately was. That report
is what turns a rule from "worked on one host" into something general.
