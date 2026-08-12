# Research: macOS Agent containment

## Question

Which currently supported macOS execution boundary can enforce Loom's Agent containment for a Pi Agent session: writable access only to its allocated Agent worktree and local session data, read-only access only to explicitly granted source snapshots, and only explicitly approved network services?

## Method and scope

Researched 12 August 2026 against Apple documentation and manual pages, Electron documentation for Electron's own boundary, and Pi documentation for Pi's own limitation. The recommendation below is an implementation decision to validate; it is not a claim that a VM configuration is safe without the test gates at the end of this note.

## Decision-ready conclusion

Use a **per-Agent-session Linux VM** launched through Apple's Virtualization framework as Loom v1's containment candidate. Run Pi and every command it starts *inside* that guest; do not run Pi as a host child process with only a `cwd` restriction. Give the guest exactly these devices:

- one read-write VirtioFS share for the allocated Agent worktree;
- one read-write VirtioFS share for that session's local data only;
- zero or more distinct read-only VirtioFS shares for approved, commit-pinned source snapshots;
- no host credential, keychain, App Group, primary-checkout, parent-Project, sibling-worktree, home-directory, clipboard, USB, audio, or graphics device;
- no network device by default; and
- a small, fixed host/guest control protocol over a dedicated Virtio socket, not a guest shell on the host.

Apple supports Linux virtual machines on both Apple silicon and Intel Macs, and its framework exposes the guest's configured devices rather than the host as a general process environment. It also requires the signed `com.apple.security.virtualization` entitlement and exposes availability/configuration validation APIs. [Virtualization](https://developer.apple.com/documentation/virtualization) · [Virtualization entitlement](https://developer.apple.com/documentation/virtualization/adding-the-virtualization-entitlement-to-your-project) · [Configuration validation](https://developer.apple.com/documentation/virtualization/vzvirtualmachineconfiguration)

This is a v1 **research result, not a release approval**. Loom must not start unattended implementation Agent sessions until the adversarial gates below pass on every supported macOS/CPU combination and packaged build. If a Device cannot create the validated VM, it cannot schedule implementation work.

## Compared approaches

| Approach | What the primary sources establish | Fit for Loom |
| --- | --- | --- |
| App Sandbox plus inherited child helper | App Sandbox is a kernel-enforced entitlement boundary for files, network, and other resources. An embedded command-line tool inherits the containing app's sandbox; an inheriting helper has only `app-sandbox` and `inherit` entitlements and inherits static parent rights, not access granted after launch such as user-selected file access. | Do not use as the Pi containment boundary. A broadly authorized Electron parent would pass broad authority to Pi; a narrowly inherited helper cannot cleanly receive a distinct dynamic Agent worktree. Use App Sandbox/hardened-runtime choices to protect Loom itself where packaging permits, but not as the per-session boundary. |
| `sandbox-exec` / custom Seatbelt profile | The installed macOS manual pages mark `sandbox-exec` and `sandbox_init` as deprecated and direct developers to App Sandbox. The sandbox overview says restrictions are generally checked when acquiring resources, and an already-open file descriptor remains usable. | Rejected for v1. It is deprecated and cannot be the supported product security promise. Do not attempt to compensate with profile-string filtering. |
| Per-session Linux VM with Virtualization.framework | Apple supports guest Linux VMs and lets the host choose Virtio storage, filesystem shares, socket devices, and network devices. VirtioFS exposes only directories configured as shares and supports read-only shares. | Recommended candidate. It makes the mount/device list the capability boundary, rather than relying on Pi, a shell, an environment variable, or a path prefix. The guest image and every exposed device remain part of Loom's security-critical implementation. |

Sources: [Configuring the macOS App Sandbox](https://developer.apple.com/documentation/xcode/configuring-the-macos-app-sandbox) · [Embedding a command-line tool in a sandboxed app](https://developer.apple.com/documentation/xcode/embedding-a-helper-tool-in-a-sandboxed-app) · [App Sandbox inheritance](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/EnablingAppSandbox.html) · [Apple DTS on unsupported custom sandbox profiles](https://developer.apple.com/forums/thread/661939) · [Virtualization](https://developer.apple.com/documentation/virtualization).

## Filesystem and process boundary

`VZSharedDirectory` exposes a specific host directory to the guest and has an explicit `readOnly` property. `VZVirtioFileSystemDevice` states that the share defines the host resources exposed to the guest; it uses the effective desktop user's UID, refuses files that user cannot access, and ignores guest attempts to change host UID/GID. These are useful controls, but they do **not** prove all traversal behaviour Loom needs. [VZSharedDirectory](https://developer.apple.com/documentation/virtualization/vzshareddirectory) · [VZVirtioFileSystemDevice](https://developer.apple.com/documentation/virtualization/vzvirtiofilesystemdevice) · [VZVirtioFileSystemDeviceConfiguration](https://developer.apple.com/documentation/virtualization/vzvirtiofilesystemdeviceconfiguration)

The host must canonicalise and validate every directory before it becomes a share: the worktree must be the exact active Agent worktree; each source snapshot must be copied or materialised at its recorded commit and exposed separately as read-only; and no share path may be an ancestor of a Project, a Loom data directory, or a credential directory. The guest must have a fixed, immutable base image and a session-specific writable guest disk. Pi's session data should be on its own session share or guest disk, never in the Project share.

Do not pass host file descriptors into the guest except those belonging to the deliberate Virtio devices. This is a general design rule, not a VirtioFS security guarantee: an already-open host descriptor or a newly added device would be authority beyond Loom's declared shares.

This architecture keeps **Provider credentials** outside Agent containment. Electron obtains a model response through a narrowly specified broker capability, or supplies a single-session credential through a purpose-built guest channel only after a separately approved design proves that it cannot read host credential storage or other credentials. It must not mount a keychain, App Group, home directory, app-data directory, `auth.json`, or credential-bearing environment variables. Pi itself says it has no built-in sandbox and runs tools with the invoking user's permissions, so Pi configuration cannot replace this boundary. [Pi security](https://pi.dev/docs/latest/security)

## Network boundary

Do not attach a network device for capability profiles with no approved egress. Apple documents `VZNATNetworkDeviceAttachment` as routing guest packets through the host to outside networks; it is connectivity, not an origin allowlist. Do not use it as a substitute for Loom's approved-services rule. [VZNATNetworkDeviceAttachment](https://developer.apple.com/documentation/virtualization/vznatnetworkdeviceattachment)

When a Ticket has an approved network capability, use a narrow guest-to-host service instead of NAT. The Virtualization framework supports Virtio sockets for port-based host/guest communication. A host-controlled service may then proxy only the Ticket's approved destinations and enforce DNS resolution, TLS/origin validation, method and redirect rules, byte limits, expiry, and an audit record. A file-handle network attachment is not automatically safe either: Apple says it maps the guest interface to a datagram socket the host application configures and manages. [Virtio sockets](https://developer.apple.com/documentation/virtualization/vzvirtiosocketdevice) · [VZFileHandleNetworkDeviceAttachment](https://developer.apple.com/documentation/virtualization/vzfilehandlenetworkdeviceattachment)

The macOS App Sandbox outgoing-network entitlement is too coarse for this requirement: it allows outgoing client connections for the app, not destination-per-Ticket permission. [Network client entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.network.client)

## Electron's role and limits

Electron renderer sandboxing, context isolation, and a narrow preload bridge still protect Loom's untrusted UI. They do not constrain Pi running as a normal host subprocess. Electron states that a sandboxed renderer delegates filesystem, system changes, and subprocess spawning to the main process; the main process is therefore not the Agent containment boundary. Keep renderer sandboxing enabled, validate IPC sender and arguments, and never expose raw Node, IPC, filesystem, or process APIs to UI or Prototype artifact content. [Electron Process Sandboxing](https://www.electronjs.org/docs/latest/tutorial/sandbox/) · [Electron Security](https://www.electronjs.org/docs/latest/tutorial/security)

The Electron main process starts/stops the VM and holds host-only authority. Its VM-control protocol must accept fixed lifecycle requests and structured Pi events only; it must never interpret a guest string as a host command, host path, arbitrary socket request, or Electron IPC request.

## Required adversarial release gates

All checks run against the signed, packaged application, with a malicious Pi task and malicious code in the Agent worktree. A failure blocks unattended implementation Agent sessions.

1. **Device and signature gate.** On each supported macOS release and CPU architecture, the packaged executable has the required virtualization entitlement, `VZVirtualMachine.isSupported` is true, and VM configuration validation succeeds. A Device that fails this gate reports no implementation capacity.
2. **Process-tree gate.** Pi, shell commands, grandchildren, background jobs, and signal-handling escape attempts all run inside the same guest boundary. Stopping the session stops that guest/process tree; no host Pi, shell, or orphan survives.
3. **Host-file denial gate.** Attempts to read or write `$HOME`, the primary checkout, Project parent, sibling Agent worktrees, Loom application data, `/Library`, system keychain locations, App Group locations, SSH configuration, cloud-synced folders, and host Unix sockets fail. Inspect the guest's mounts and process environment as well as command results.
4. **Traversal and symlink gate.** `..`, absolute paths, repeated symlinks, symlinks created before/after VM start, dangling links, hard links where possible, renamed share roots, Git submodules, and worktree/snapshot links cannot escape the exposed share. This is essential because Apple documents that even App Sandbox container symlinks resolving to sensitive folders require separate authorization; do not assume a string-prefix check is a security proof. [App Sandbox file access](https://developer.apple.com/documentation/xcode/configuring-the-macos-app-sandbox)
5. **Write-scope gate.** Only the worktree and session-data shares change. Every source snapshot is read-only, and tests show failed writes, chmod/chown attempts, and remount attempts. Host-side Git metadata and the primary checkout remain unchanged.
6. **Credential gate.** The guest sees no provider credential, control-plane token, Electron cookie, keychain/App Group access, inherited host environment secret, SSH agent, or credential helper. Logs, guest disks, snapshots, and VM bundles contain none after start, pause, resume, failure, and cleanup.
7. **Network denial gate.** With no network device, DNS queries, IP sockets, loopback probes, Unix-domain host socket probes, proxy-environment use, and raw outbound connection attempts fail.
8. **Mediated-egress gate.** With an approved capability, only the exact approved origin/service works. Reject IP literals, private/link-local destinations, DNS rebinding, redirects across origins, CONNECT tunnelling, alternate ports/SNI, proxy environment variables, and requests after capability expiry. Audit every allowed request.
9. **Control-channel gate.** Malformed/oversized guest messages and Pi/tool output cannot make Electron execute a host command, broaden a mount, read a file, open an external URL, alter an Action, or mint another capability. The guest only receives the documented control protocol.
10. **Regression gate.** Re-run all gates after macOS, Electron, guest-image, Pi, virtualization-framework, or filesystem-sharing changes. A security regression disables automatic scheduling until fixed.

## Implementation consequences

- Build the VM runner, immutable guest image, share preparation, guest control service, and adversarial harness before promising implementation-profile automation.
- Declare supported macOS versions/architectures only after their gates pass. Apple documents both Apple-silicon and Intel Linux guests, but Loom's release support remains a tested-product decision rather than an inference from that API. [Creating and Running a Linux Virtual Machine](https://developer.apple.com/documentation/virtualization/creating-and-running-a-linux-virtual-machine)
- Use a capability profile to decide *which* share/device/service may exist. A Project-read capability adds only an individually named, read-only source snapshot; it never adds a Project root.
- Fail closed: if VM provisioning, device validation, containment attestation, or a required test is unavailable, Loom leaves the Ticket ready/waiting for an explicit human-managed alternative and does not launch an unattended implementation Agent session.

## Sources

- [Apple: Configuring the macOS App Sandbox](https://developer.apple.com/documentation/xcode/configuring-the-macos-app-sandbox)
- [Apple: Embedding a command-line tool in a sandboxed app](https://developer.apple.com/documentation/xcode/embedding-a-helper-tool-in-a-sandboxed-app)
- [Apple: Enabling App Sandbox inheritance](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/EnablingAppSandbox.html)
- [Apple DTS: custom sandbox profiles are unsupported for third-party products](https://developer.apple.com/forums/thread/661939)
- [Apple: Virtualization framework](https://developer.apple.com/documentation/virtualization)
- [Apple: Adding the Virtualization entitlement](https://developer.apple.com/documentation/virtualization/adding-the-virtualization-entitlement-to-your-project)
- [Apple: Virtio filesystem device](https://developer.apple.com/documentation/virtualization/vzvirtiofilesystemdevice)
- [Apple: Shared directory](https://developer.apple.com/documentation/virtualization/vzshareddirectory)
- [Apple: NAT network attachment](https://developer.apple.com/documentation/virtualization/vznatnetworkdeviceattachment)
- [Apple: Virtio socket device](https://developer.apple.com/documentation/virtualization/vzvirtiosocketdevice)
- [Apple: File-handle network attachment](https://developer.apple.com/documentation/virtualization/vzfilehandlenetworkdeviceattachment)
- [Electron: Process Sandboxing](https://www.electronjs.org/docs/latest/tutorial/sandbox/)
- [Electron: Security](https://www.electronjs.org/docs/latest/tutorial/security)
- [Pi: Security](https://pi.dev/docs/latest/security)
