# Contributing

This repository follows the OzarLand community contribution guide:
[CONTRIBUTING.md in `ozarland`](https://github.com/OzarLand/ozarland/blob/main/CONTRIBUTING.md).

In short:

1. Give context: OpenStack release, components, deployment method, environment conditions, what you checked.
2. No sensitive data: no secrets, real hostnames, domains, IP or MAC addresses, customer or person names,
   unredacted logs ([how to redact](https://github.com/OzarLand/ozarland/blob/main/docs/conventions/sensitive-data.md)).
3. Flag risky commands: ⚠️ state-changing, 🛑 destructive
   ([risk warnings](https://github.com/OzarLand/ozarland/blob/main/docs/conventions/risk-warnings.md)).
4. Say where it was observed; never present a procedure as universal.
5. Use the templates in `templates/`, open a pull request, sign off your commits (`git commit -s`).

Repository-specific notes:

<!-- Anything specific to this repository: directory to use, naming of pages, extra fields. Keep it short. -->
- Put a pattern in `topologies/`, a network or storage design in `networking/` or `storage/`, a layout in
  `configuration/`, a solved deployment error in `deployment-errors/`, a day-2 procedure in `operations/`,
  release-specific notes in `version-notes/`, and side-by-side comparisons in `comparisons/`.
- Name the file after the pattern or symptom, lowercase, hyphenated.
- Add the page to the index table in `README.md` in the same pull request.
