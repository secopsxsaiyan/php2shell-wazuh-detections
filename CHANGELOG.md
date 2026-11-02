# Changelog

All notable changes to this pack are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project uses
[semantic versioning](https://semver.org/spec/v2.0.0.html) with detection
content read as follows:

- **major**: a rule id changes, a rule is removed, or a condition narrows in a
  way that could stop an existing deployment from alerting.
- **minor**: a rule is added, or a condition widens to catch more.
- **patch**: documentation, tuning notes, or CI.

## [1.0.0] - 2026-11-01

### Added

- `rules/php2shell_shell_spawn.xml`, rule **102007** at level 12: a shell
  `execve` by a non-root account, which is the observable end of successful
  exploitation. Requires auditd `execve` events reaching the manager.
- `rules/php2shell_apache_oom.xml`, rule **102005** at level 10: the
  Apache-wrapped `[php:error] PHP Fatal error` line, which stock 30301 decodes
  at level 0 so nothing alerts by default.
- `sigma/php2shell.yaml`: Sigma equivalents of both detections, validated with
  `sigma check`.
- CI validating the XML, rule structure, the Sigma rules, pack hygiene, and
  that every rule id is documented.

### Notes on this release

- Validated on one configuration: PHP 8.5.11 with Apache `mod_php` on Debian,
  Wazuh 4.14.7. A patched host was not tested.
- The uid condition on 102007 is load-bearing. Without it, an idle host with no
  attack running produced 40 level-12 alerts in a few minutes, all uid 0.
- No access-log rule is included. The exploitation attempt returns HTTP 200 and
  Apache Combined carries no request body, so the only matching access-log rule
  would be "POST to a PHP URL", which fires on ordinary traffic.

[1.0.0]: https://github.com/secopsxsaiyan/php2shell-wazuh-detections/releases/tag/v1.0.0
