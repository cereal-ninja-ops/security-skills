CI/CD Supply-Chain Defense (trust provenance of inputs, not green checks)
When this applies
Hardening, threat-modeling, or reviewing a build/release/deploy pipeline. Use it to turn "the build passed and it's signed" into a set of trust decisions that actually hold against a motivated attacker who already has some foothold (a leaked token, commit access, or control of a dependency).

The one principle everything derives from
Almost every pipeline control guards the definition or identity (which branch, which key, which pipeline file), not the contents or the exceptions (what the build actually executes, what rides in the tarball, which lanes skip review). A signature, a green check, and a passing build attest who produced something and that it ran — none of them attest that the inputs were clean. Defense = make trust decisions depend on the provenance of inputs, and close every gap where code can enter after review.

Attack class → the control that actually stops it
Secrets committed then “deleted” from history. Rewriting history does not remove the object; it stays reachable by hash. → Secret scanning at commit time and in CI; rotate on exposure (assume leaked = burned); short-lived, narrowly-scoped tokens so a leak reaches little; never treat a history rewrite as deletion.

Poisoned Pipeline Execution (PPE). The pipeline runs repo-controlled scripts (make test → tests/run.sh) with secrets in the environment; whoever edits the script runs code as CI. → Never inject deploy/signing secrets into a stage that runs attacker-controllable scripts; least-privilege CI identity (run untrusted build/test as an unprivileged user with no path to master secrets); separate the protected pipeline definition from the unprotected scripts it calls, and treat those scripts as untrusted.

Branch-protection bypass via an exempt lane. Protection is a policy keyed on ref names; automation/hotfix/tag lanes are deliberate holes. → Audit the exceptions, not just the protected branch. An exempt lane that skips review is as powerful as the branch it relieves — and is often the same lane that can deploy. Minimize review-skipping lanes; make sure no exempt lane can reach production.

Build-time / release-time injection (artifact ≠ source). Payload is staged at build time (host-only build machinery, a generated file, an encoded fixture) and never appears in a reviewed commit. → Reproducible / hermetic builds: build in a clean environment with no host-only machinery able to inject, then diff the artifact against a from-source build — any delta is the attack. Review build scripts and test fixtures as code. Diff the release tarball against the git tree (tar tzf vs git ls-files).

Content-blind signing abuse. The signer certifies whatever the build emits; a valid signature over a malicious artifact still verifies. → Require provenance attestation (SLSA / in-toto), not just a valid signature. The verifier must check “built from reviewed source via trusted steps,” not merely “signed by the pipeline.” (This is also why signed commits don't help: they attest authorship, and the payload isn't in a commit.)

Self-shipping deploy to the fleet. One signed artifact auto-deploys everywhere, trusted because it's signed and came through the pipeline. → Gate deployment on provenance at admission (only artifacts whose attestation traces to reviewed main may deploy); require independent approval for fleet-wide rollout; sandbox/VM at runtime as defense-in-depth (contains blast radius — but note it limits damage, not trust, so it's a last layer, not the fix).

Review checklist (fast pass on an unfamiliar pipeline)
What does each stage execute, and is any of it repo-controlled? Does that stage hold secrets?
Which identity runs the build? Can it read deploy/signing/master secrets?
What branch/tag namespaces bypass review? Can any of them deploy?
Does the shipped artifact get verified against source, or just signed?
Does anything trust “signed” as “safe”? Where does provenance actually get checked?
Is there any build machinery that lives outside the repo (release host, generated files, vendored blobs, dependency install hooks)?
Mantra
A green check says it ran. A signature says who made it. A passing review covers what was reviewed. None of them covers what the build injected or which lane skipped the review. Trust inputs' provenance, verify artifact-equals-source, and audit the exceptions.

Real-world anchors
xz/liblzma (CVE-2024-3094, 2024): malicious stages lived in release tarballs' build files and a binary test fixture, absent from git — the artifact≠source gap.
SolarWinds Orion (2020): a compromised build produced backdoored updates signed with the genuine code-signing cert — signing attested provenance, not a clean build.

