# Wazuh and Sigma detections for PHP2Shell (CVE-2026-109618 / CVE-2026-109619)

[![validate](https://github.com/secopsxsaiyan/php2shell-wazuh-detections/actions/workflows/validate.yml/badge.svg)](https://github.com/secopsxsaiyan/php2shell-wazuh-detections/actions/workflows/validate.yml)

Two detections for PHP2Shell against Apache `mod_php`, validated on Wazuh
**4.14.7** and PHP **8.5.11**:

| Rule | Signal | Level |
|---|---|---|
| `rules/php2shell_shell_spawn.xml` (102007) | a shell `execve` by a non-root account, which is how exploitation ends when it succeeds | 12 |
| `rules/php2shell_apache_oom.xml` (102005) | the Apache-wrapped `[php:error] PHP Fatal error: Allowed memory size` line | 10 |

Sigma equivalents are in `sigma/php2shell.yaml`. All original wording, MIT.
No PoC source, payloads, or request builders here. Research links:
[ikow/php2shell-poc](https://github.com/ikow/php2shell-poc),
[devploit/php2shell](https://github.com/devploit/php2shell).

## The finding worth your time

The obvious rule is "alert when a web worker executes a shell." Written the
obvious way, it is unusable.

On an **idle** test host with **no attack running**, the version without a uid
condition fired **40 times in a few minutes**. Every match was uid 0,
`exe /usr/bin/dash`, `a0` of `sh` or `/bin/sh`, mostly a single ppid: cron,
systemd, and the Wazuh agent's own wodles running `sh -c`. At level 12 that
buries the one event the rule exists to catch.

So 102007 requires a **non-root** uid. That single condition is the difference
between a detection and an alert generator, and the naive form looks correct
under `wazuh-logtest`.

## What the analysis showed

Confirmed code execution produced one alert at uid 33 (`www-data`). The rule
keys on the worker's `execve`, not on any exploit output:

```
uid 33 | a0 sh | a1 -c | exe /usr/bin/dash | level 12
```

| Check | Result |
|---|---|
| Idle host, no attack, uid-less rule | 40 alerts, all uid 0 |
| Same host with the uid condition | 0 alerts |
| Confirmed execution | 1 alert, uid 33, level 12 |
| Captured audit line as recorded (uid 0) | no match |
| Same bytes with uid 33 | match at level 12 |

The last two rows are one event with one field changed, which is the clearest
statement of what the rule keys on.

The memory-exhaustion path is a separate signal with no shell. The response on
the wire was **HTTP 200**, and stock Wazuh stayed quiet: 31122 keys on web 500,
31420 expects a bare `PHP Fatal error` rather than the Apache `[php:error]`
form, and 30301 and 31108 both decoded the events at level 0. Collection was
never the problem. The level was.

There is deliberately **no access-log rule** here. The exploitation attempt
returns 200, and Apache Combined carries no request body, so the only
access-log rule that would match is "POST to a PHP URL", which fires on
ordinary traffic. The error log is where this is visible.

Runs that did not achieve execution produced **no** 102007 at all, so a failed
attempt does not generate a false shell detection.

## Limits of this testing

- Both rules were validated on **one** configuration: PHP 8.5.11 with Apache
  `mod_php` on Debian, Wazuh 4.14.7. Upstream states affected is 5.4.0 through
  8.5.11, fixed in 8.2.35 / 8.3.36 / 8.4.27 / 8.5.12.
- A **patched** host was not tested, so these rules are not evidence that a
  patched build is silent.
- 102007 needs auditd `execve` events reaching the manager. Without them there
  is nothing to match.
- 102007 will miss execution that spawns a non-shell child, or that reaches
  root before spawning.
- On a general-purpose host a legitimate non-root shell is a false positive.
  102007 suits single-purpose web hosts; see the tuning note in the file.
- No `zend_mm_heap` corruption line was observed, though the Sigma rule covers
  it.

## Install

1. Copy both files from `rules/` into your Wazuh custom rules directory, for
   example `/var/ossec/etc/rules/`.
2. Change the rule ids if 102005 or 102007 collide with your ruleset.
3. Run `wazuh-analysisd -t`, then restart the manager. On 4.14.7
   `wazuh-control` has no `reload` action, only `restart`.
4. Confirm auditd on your PHP hosts records `execve` and ships it, or 102007
   stays inert. To check without enabling archiving, deploy a copy of the rule
   at level 3, run `sudo -u www-data sh -c true` on the host, and look for it
   in `alerts.json`. Remove the copy afterwards.
5. Read the tuning note at the top of each file before deploying broadly.

## Validating and contributing

CI checks every push: `xmllint`, a structural pass over the rules (unique ids, a
level and description on each, patterns that compile, and no rule whose only
condition is its parent), `sigma check` from sigma-cli, an ASCII-only check, and
a check that every rule id is documented here.

The numbers above come from a live deployment, not a simulation. An agent on
the target shipped auditd events to a Wazuh 4.14.7 manager, the exploitation
attempts were real runs of the published entrypoint, and the alert counts are
the manager's: forty on an idle host before the uid condition, then one at uid
33 on the run that achieved execution, taking 102007 from 40 to 41 with every
earlier hit being uid 0. The uid-0 against uid-33 comparison was run through
`wazuh-logtest` on that manager using the real captured audit line rather than
a synthetic one.

On your own manager, `wazuh-logtest` is the authority: paste a sample line and
confirm the rule you expect is the one that fires.

Version **1.0.0**. Changes are recorded in [CHANGELOG.md](CHANGELOG.md), where a
rule id change or a narrowed condition counts as a major version because it can
stop an existing deployment from alerting.

[CONTRIBUTING.md](CONTRIBUTING.md) has the bar a new rule has to meet. A
false-positive report from your estate is more useful than a new rule: these
detections were validated on one configuration, and yours is information this
project cannot generate on its own.

## License

MIT. See `LICENSE`.
