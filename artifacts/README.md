# Restore the paused ServerDesk session

Download `serverdesk-session-5da96b3.tar.gz` and `serverdesk-session-5da96b3.tar.gz.sha256` from this folder. Save both files in the same directory.

On the destination Linux machine, run these commands from that directory.

```sh
sha256sum -c serverdesk-session-5da96b3.tar.gz.sha256
tar -xzf serverdesk-session-5da96b3.tar.gz
```

Read `serverdesk-session-5da96b3/README.md` before restoring the repository. It describes the required tools, a new destination directory, and environment setup. Do not extract the source snapshot over an existing checkout.

The package contains Git history through checkpoint `5da96b38ac502fc9daa9730b16c6c2c3ffebfde3`, the saved working tree, specifications and tickets, a session handoff, selected verification evidence, and Xstack tool source. It excludes credentials, the live chat, running processes, and build output.

The implementation is unfinished. Known lint failures, unverified changes, and external integration blockers remain recorded in `SESSION-HANDOFF.md` inside the package. This PR transfers that state. It does not accept tickets or merge the unfinished application code into the default branch.

The transfer smoke check extracted the package into a temporary directory, verified all 11 payload checksums, restored the Git checkpoint, and matched 443 regular files, 68 symlinks, and the saved tracked user changes. Application tests were not run for this packaging task.

Checksums detect corruption. They do not authenticate the sender. Review the restored project and configure the new machine before running application or workflow commands.
