# 00: Overview

**Status:** reference
**Contract:** `.scratch/adr-0001-openssh-transport/spec.md`, section `# Design`.

## Files

These files describe planned implementation. At planning base `5b62b8d`, the executable only prints a greeting. ADR-0006:01 introduces shared declarations in their canonical owner modules; these tickets implement the workflows without redefining foreign records.

```
src/
├── application/
│   └── connection_manager.rs [01, 02, 03, 04, 05] early borrowed projections and sole master/channel owner
├── domain/
│   └── connection.rs [02] Connection and Terminal domain observations
├── infrastructure/
│   └── ssh/
│       └── mod.rs [01, 02, 03, 04, 05] actual OpenSSH, askpass and confined orphan cleanup
└── ui/
    └── transport.rs [02, 05] private main-thread GTK authentication/VTE endpoint
tests/
├── support/
│   └── transport.rs [01, 02, 03, 04, 05] isolated actual transport fixtures, never fake transport
├── transport_gtk.rs [02, 05] future actual GTK/VTE keyboard and lifecycle journeys
└── transport_workflows.rs [01, 02, 03, 04, 05] future actual Incus/OpenSSH workflow and narrow invariant tests
```

## Architecture

```
E6 preadmission -- StartupOwnership/RuntimeDirectory --> SSH cleanup [01]
SSH cleanup -- CleanupReport --> E6 admit -- ReadyOwnership --> manager [02]
E2 configuration + E3 trust -- CompiledConfiguration/PreparedStart --> manager [02,04]
manager -- TransportUi/TerminalUiCommand --> GTK VTE [02,05]
GTK VTE -- TerminalLifecycleReport --> manager [02,05]
E4/E5/E3 -- ConsumerRequest/ConsumerTarget --> manager -- Consumer/ConsumerEvent --> owner [03,04]
manager -- ConnectionChanged/ChannelEnd --> GTK + workflow owners + E6 [02,03,05]
```

The manager is the only master/session writer; GTK alone owns VTE/widget state and reports exact child/PTY facts.
Configuration/trust validate their own sealed material; transport never treats views as launch authority or SSH exit as administrative proof.
ADR-0006:01 supplies common declarations and pure Store projections, not future domain authority; real primary admission and each runtime issuer must precede its consumer.

## Order

- Bootstrap safety: ADR-0006:01 → 01 → ADR-0006:02 → 02.
- Direct Connect/authentication/independent terminals: 01 → 02.
- Typed consumers and registered routes: 02 → 03 → 04.
- Disconnect/master loss/shutdown: 02 → 03 → 04 → 05.
- No local work tickets run in parallel; shared-owner continuations are concurrent runtime work, not independent ticket prerequisites.

- 01: Clean actual orphan transport before admission; local blockers none.
- 02: Connect and authenticate one shared master with independent terminals; local blockers 01.
- 03: Carry isolated typed SFTP Operation and Collector consumers; local blockers 02.
- 04: Traverse registered Jump routes and discover through trusted prefixes; local blockers 02, 03.
- 05: Drain every descendant and integrate master loss and shutdown; local blockers 02, 03, 04.

## In plain words

First, the app clears away its old connections safely instead of reusing leftovers. Then you can connect with your normal SSH authentication and open separate terminals that keep their own input and Server context. File transfers, administration and monitoring share that login without mixing their data. Registered Jump Servers provide the only allowed route to a remote Server, and each connection still needs its own trust decision. Closing a terminal releases that shell, while disconnecting closes every part of that shared connection before ending the login. Other connections keep working, and failures tell you what was actually observed rather than pretending a remote change succeeded. The desktop stays usable with the keyboard while those steps run. Approved T3 and L1 govern trust, while file policy remains blocked and real producers still need implementation.

## Comments

E1, T3, L1 and S decisions are approved. Numbered implementation assignments are ready-for-agent after their actual listed producer prerequisites, not runtime-complete. T3 permits accepted VERIFIED, ADOPTED and explicitly committed UNVERIFIED for Direct use, VERIFIED only for every Jump role, and blocks NONE/MISMATCH in both. Fresh preflight, original binding and atomic SQLite snapshot remain required. File-policy 03 stays independently unresolved; its downstream enabled scenarios cannot be claimed delivered by read-only transport or monitoring.
