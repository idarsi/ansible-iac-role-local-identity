# Contributing

Keep `iac_blueprint.local_identity` compatible. New fields must be validated,
normalized, documented, and tested in the same change. Do not add sudo package
installation to the role. Destructive behavior requires an explicit guardrail,
and normal convergence must remain idempotent.

Before proposing changes, run the syntax check, production-profile
`ansible-lint`, and applicable Molecule scenarios. Never add real passwords or
private keys to examples.
