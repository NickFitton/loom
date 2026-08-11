# Define the account, device, and authentication contract

Type: grilling
Status: resolved
Blocked by: none

## Question

How do Better Auth, the Electron client, and NestJS establish email/password sessions; register, name, and revoke Devices; protect device-local provider credentials; and prevent a revoked or stale device from executing or synchronizing Loom state?

## Answer

Loom uses Better Auth's supported Electron sign-in flow: the Electron application starts the sign-in, opens the system browser for email/password registration or sign-in, and receives a one-time PKCE-protected handoff on its registered custom deep link. Electron main owns the Better Auth client, session material, and the narrow renderer IPC bridge; the renderer never receives a reusable authentication token or provider credential.

### Account registration and remembered sign-in

- Email/password registration creates an unverified registration. Until email verification completes, Loom shows only the verification, resend, and sign-out flow: it creates no usable Device, Project access, Agent session, or control-plane authority.
- After verification, Loom establishes a remembered Better Auth session in Electron main. Its persistent session material is protected by OS-backed secure storage; it is distinct from the shorter Device lease described below.
- Password reset revokes every Device. A person must sign in again on each computer they still want to use.

### Devices and credentials

- The first verified sign-in on a computer automatically enrols a new Device. Loom creates a Device key pair in secure storage, proposes an editable name, and records first- and last-seen times.
- Account settings list the enrolled Devices and let a person rename or revoke any non-current Device. A Device cannot revoke itself.
- A Device revocation is terminal for that Device identity. It invalidates its control-plane authority, stops its work when the app receives the revocation, deletes its local Device key and provider credential, and rejects all later synchronization. A later verified sign-in on that computer enrols a distinct new Device.
- Signing out is not revocation: it stops local work and removes the current app session, but retains the enrolled Device and its protected provider credential for the next verified sign-in.
- A provider credential and Device private key never leave the Device's OS-backed secure credential store. If secure encrypted storage is unavailable, Loom may show non-execution account information but must not persist provider credentials or run Agent sessions.

### Device authority and stale execution

- A control-plane-issued Device lease, separate from the remembered Better Auth session, gives a Device authority to start, resume, execute, and synchronize. The lease is valid for at most 30 minutes and Loom attempts renewal roughly every 5 minutes while connected.
- Loss of connection does not revoke the Device. It may continue local work until the lease expires. On expiry, Loom pauses active Agent sessions and blocks starts, resumes, and synchronization until it revalidates with the control plane.
- A received revocation takes effect immediately: the execution plane pauses active Agents and blocks all synchronization. The control plane independently rejects every request from a revoked Device or one with no valid lease, so an offline or stale client cannot later submit authority-bearing changes.
- The orchestration and recovery protocol must use the lease when it reconciles local session state; it owns the ordering and idempotency details of rejected or resumed reports.
