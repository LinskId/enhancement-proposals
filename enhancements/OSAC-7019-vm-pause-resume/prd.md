# VM pause/resume

| Field | Value |
|-------|-------|
| Author(s) | ibengal |
| Jira | [OSAC-7019](https://redhat.atlassian.net/browse/OSAC-7019) |
| Date | 2026-10-30 |
| Target milestone | 0.5 |
| Status | Draft for review |

## Problem Statement

Tenant Users need to temporarily freeze a running workload and later continue
its existing execution — for example, to hold an in-memory application state
during a short maintenance window without shutting the guest down. OSAC's
current VM lifecycle interface provides Start, Stop, Restart, and Delete;
tenant pause/resume is the missing capability described in
[OSAC-1844](https://redhat.atlassian.net/browse/OSAC-1844). Stopping a VM
loses its running application and RAM state, so it cannot satisfy that need.

Pause is not a capacity- or cost-reclamation feature: while paused, RAM and
existing resource reservations remain allocated on the host and continue to
count against tenant quota. Reclaiming capacity requires Stop (or a future
hibernate capability, which is out of scope). CPU/memory configuration changes
that require a reboot continue to use today's RestartRequired / restart path;
pause does not replace that upgrade flow.

## In Scope

- VMaaS ComputeInstance pause/resume through the tenant UI and API. Pause
  freezes guest execution and I/O while retaining RAM; Resume continues that
  execution without a new boot. [Clarify: R1.Q3]
- Pause/resume interactions with Start, Stop, Restart, and Delete, including
  overlapping requests and invalid states, as specified in the acceptance
  cases below.
- Automated validation and E2E coverage of UI and API journeys, retained RAM,
  resource persistence, failures, and tenant isolation.

## Out of Scope

- Scheduled or automated pause/resume.
- Dedicated Pause/Resume commands in the OSAC CLI for milestone 0.5; the API
  remains available for automation. [Clarify: R14.Q1]
- Suspend-to-disk, hibernation, or saving RAM to persistent storage.
- Live migration between hosts or clusters.
- A new tenant quota dashboard or quota-management capability.
- Admin-specific pause/resume views or permissions beyond existing VM lifecycle
  access.
- Changes to existing lifecycle behavior outside the pause/resume interactions
  above, including Delete during an ordinary Start or Restart.
- Billing-rate or metering-pipeline changes for paused VMs; any paused
  billing-display work remains owned by
  [OSAC-1844](https://redhat.atlassian.net/browse/OSAC-1844).
  [Clarify: R16.Q1 / D29]

## User Stories

### Tenant Admin / Tenant User

1. **Pause execution.** As a Tenant Admin or Tenant User, I want to pause a
   running VM so that its workload stops executing while its application
   state stays in memory.

   Acceptance: through either the UI or API, an authorized user can pause a
   running VM. When the operation succeeds, the workload no longer advances
   its execution and the VM is reported as paused.

2. **Resume execution.** As a Tenant Admin or Tenant User, I want to resume a
   paused VM so that my application continues from its retained state.

   Acceptance: an application holding a value only in RAM retains that value
   across pause/resume and can continue its computation. The guest has the
   same boot identity, and OSAC continues the same VM instance without
   replacing it.

3. **Retain the VM's resources.** As a Tenant Admin or Tenant User, I want my VM
   resources preserved across pause/resume so that I can continue using the
   same configured workload.

   Acceptance: the VM identity, configuration, attached disks, and previously
   persisted disk contents, assigned IP address, and guest MAC address are
   unchanged after a successful pause/resume, including a long pause.
   Retaining VM memory does not guarantee that external connections or sessions
   remain valid. [Clarify: R2.Q1]

4. **Observe progress and state.** As a Tenant Admin or Tenant User, I want to
   see the progress of my pause/resume request so that I can tell when the VM
   has reached the requested state.

   Acceptance: the UI and API distinguish running from paused, show progress,
   and report the observed final state. The UI disables opposing pause/resume
   actions during transitions. Lifecycle outcomes match the existing Start/Stop
   desired-state pattern (not raw KubeVirt imperative reject-on-repeat
   behavior). [Clarify: R15.Q1 / D30]

   - Pause on Paused, Resume on Running, a repeated pending request, and Start
     on Paused or during Pause/Resume succeed without new work. For overlapping
     Pause/Resume, the latest accepted request determines the final state;
     intermediate states need not complete.
     [Clarify: R3.Q1, R4.Q1, R6.Q3, R7.Q2, R7.Q3]
   - Pause/Resume on Stopped, Failed, or deleting, and Resume during Restart,
     return invalid-state errors. Pause on Stopped saves no future request;
     the UI offers neither action on Stopped or Failed. On Failed, the existing
     reason stays visible. [Clarify: R6.Q2, R7.Q4, R8.Q4, R9.Q2, R10.Q1]
   - Pause during Start/Restart takes effect after a successful boot; a boot
     failure is shown instead of Paused. Stop/Restart on a paused VM or during
     Pause/Resume supersedes those requests and discards RAM. Stop reaches
     Stopped; a later Start or a Restart boots unpaused. [Clarify: R5.Q1,
     R5.Q2, R6.Q1, R7.Q1, R8.Q1, R8.Q2, R8.Q3]
   - Delete on a paused VM or during Pause/Resume is accepted without waiting.
     It supersedes those requests, removes the VM and retained RAM, and cannot
     be undone by a pending Pause, including one accepted during Start/Restart.
     [Clarify: R9.Q1, R9.Q2, R9.Q3]

   If the host is unreachable, Paused is shown only as the last known,
   unverified state. Confirmed loss of that VM instance removes Resume;
   the UI and API report a new boot as Running only after it succeeds, or
   Failed if it does not, and explain that the retained RAM was lost.
   [Clarify: R11.Q1]

5. **Understand failure.** As a Tenant Admin or Tenant User, I want a failed
   pause/resume request to explain the failure so that I can decide what to do
   next.

   Acceptance: a failed request produces an error associated with the
   requested operation. The reported VM state reflects the observed outcome,
   including whether it is running or paused; the operation error does not
   itself change the reported state to Failed. If the VM is paused, Resume
   remains available.
   [Clarify: R10.Q1]

6. **Protect tenant workloads.** As a Tenant Admin or Tenant User, I want
   pause/resume access restricted to authorized VMs so that another tenant
   cannot interrupt my workload.

   Acceptance: Pause and Resume follow the same VM authorization as Start and
   Stop. The API rejects a request outside the caller's permitted tenant scope
   without changing the VM. The UI offers actions to the same users who can
   use existing VM lifecycle controls. [Clarify: R13.Q1]

7. **Understand retained resources.** As a Tenant Admin or Tenant User, I want
   to understand the resources my paused VM retains so that I can make an
   informed decision about pausing it.

   Acceptance: user and API documentation covers actions, permissions, states,
   errors, external-session limitations, and the distinction from stop/start
   and hibernation. It explains that RAM and existing resource reservations
   remain allocated while paused and states the IP/MAC retention guarantee.
   Any applicable tenant quota or allocation report continues to count those
   reserved resources; pausing does not increase available quota.
   [Clarify: R12.Q1]

## Dependencies

- **KubeVirt:** its documented
  [pause/unpause behavior](https://kubevirt.io/user-guide/user_workloads/lifecycle/#pausing-and-unpausing-a-virtual-machine)
  supplies the execution and memory-retention semantics (CPU/I/O freeze with
  RAM retained). It reports Paused, or Unknown when the host state cannot be
  obtained, but does not reconcile a desired pause state for an existing VM.
  OSAC exposes the Start/Stop-aligned lifecycle outcomes above; the design
  adapts KubeVirt's imperative pause/unpause API to those outcomes
  (including no-ops and latest-wins). [Clarify: R15.Q1 / D30]
  The design must validate supported VM configurations and document that a
  paused VMI remains scheduled on its node (subject to ordinary
  Kubernetes/CNV node-pressure and eviction behavior outside this Feature).
- **VMaaS networking:** preservation must align with the accepted
  [networking decisions](https://github.com/osac-project/osac/blob/main/docs/agent-context/networking-decisions.md)
  and [VMaaS networking design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1435-vmaas-networking/design.md#network-status-and-address-discovery).
  The design must validate the pause-specific IP/MAC guarantee on supported
  networks, including long pauses, and accurate OSAC state reporting.
- **OSAC-UI and OSAC-VMaaS:** UX/UI design and the lifecycle capability must
  agree on actions, progress, errors, and tenant permissions.
  [OSAC-7019](https://redhat.atlassian.net/browse/OSAC-7019) owns pause/resume
  lifecycle UI and API. [OSAC-1844](https://redhat.atlassian.net/browse/OSAC-1844)
  owns any paused billing-display expectations. [Clarify: R16.Q1 / D29]
- **Deployment compatibility:** OSAC does not currently support in-place
  platform upgrades. The design must assess compatibility for existing VM
  state and independently deployed components, as required by
  [OSAC's platform scope guidance](https://github.com/osac-project/osac/blob/main/docs/agent-context/osac-dimensions.md#milestone-scoping).

---

## Provenance

Authored: draft @ prd 0.11.5 - 7e2d081, workspace main @ fa782878f
Final: revise @ prd 0.11.5 - b369da0, workspace main @ fa782878f

> Context changed between draft and revise.

Review follow-up: locked Q8/Q9 and strengthened Problem Statement (capacity
vs freeze, restart-for-resize) from PR #450 technical review.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.5","ai_workflows":"b369da0","source_repo":"fa782878f","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
