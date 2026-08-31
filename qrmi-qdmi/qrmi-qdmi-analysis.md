# QRMI And QDMI Comparison

This document captures the QRMI and QDMI comparison being performed with the
QFw shim QPM service as the test vehicle. The goal is to compare the public
runtime and resource interfaces exposed by QRMI and QDMI, then use QFw tests to
observe how those interfaces behave when placed behind one QPM-facing contract.

The comparison follows the API-category direction discussed in
`openQSE/openqse-spec` discussion 31. A quantum resource interface has several
consumers. Applications, resource managers, schedulers, operators, monitoring
services, and authentication services do not need the same calls. The axes below
split the interface by function so each category can be evaluated on its own.

## Versions Under Comparison

Both libraries are moving quickly, and much of what follows is behavioural
rather than architectural — error propagation, what a payload contains, what an
implementation costs. Those observations are only meaningful against a stated
version, and several are expected to change. Anything recorded here should be
re-checked before being carried into a specification decision.

Unless an entry says otherwise, observations were made against:

| Component | Version | Role |
|---|---|---|
| `qrmi` | 0.24.0 | QRMI interface and its IQM resource implementation |
| `iqm-qdmi` | 1.4.0 | QDMI-on-IQM, the QDMI device implementation for IQM |
| `mqt-core` | 3.9.2 | QDMI client, headers, and the FoMaC layer the Python caller uses |
| `QDMI` | 1.3.3 on both sides | the specification each side was built against. They were one patch release apart until iqm-qdmi 1.4.0, and that gap was load-bearing. See below |
| `iqm-client` | 34.0.1 | IQM Server client used by the QFw drivers |
| `iqm-station-control-client` | 12.1.1 | IQM station-control models |
| `iqm-pulse` | 13.0.1 | IQM circuit objects produced by transcoding |
| `qhw-iqm` | 0.1.0 | normalization of IQM data into `qhw-*` records |

Hardware observations are against the ORNL IQM 20-qubit device, reached through
the QFw shim (`openQSE/QFw`, `services/svc_lib_qpm`).

**These versions replace an earlier baseline of `qrmi` 0.17.2, `iqm-qdmi` 1.2.0,
and `mqt-core` 3.7.0, and the axes below have been rechecked against them.**
Most findings survived. Where one did not, the axis says what changed rather
than quietly presenting the new state as though it had always held. Four
findings moved materially: the calibration set identity under Device
Introspection and Calibration And Quality Data, queue position under Telemetry,
the not-supported answer under Extensibility And Versioning, and most of the
Error Model axis, three of whose central claims have been fixed upstream.

The QDMI row records a version for each side because for part of this recheck
they differed. The client, MQT Core 3.9, was built against QDMI 1.3.3 while the
device library, QDMI-on-IQM 1.3.0, was built against 1.3.2. One patch release
apart, and directly observable from the caller. **iqm-qdmi 1.4.0 closed that gap
by moving to 1.3.3**, so the two sides now agree and the specific failure is no
longer reproducible on this stack.

It is still recorded under Extensibility And Versioning, and deliberately so. A
defect that a version bump made disappear is not the same as a defect that was
never real, and independently released device libraries lagging the client is
the normal condition for the ABI model QDMI is built on rather than an accident
of this measurement. The next such gap will behave the same way.

On source citations. Where this document cites QDMI-on-IQM source it means the
corresponding upstream tag, not any local working tree — `iqm-qdmi` ships as a
wheel containing a compiled device library, so a checkout used for reading is
not necessarily the code that ran.

Citations are anchored by symbol or function name rather than by line number
wherever the file is one that moves. Line numbers were tried first and did not
survive: every QDMI-on-IQM line reference in Calibration And Quality Data had
drifted by the time it was rechecked at 1.2.0, and the QFw ones had been moved
by later commits to those same files.

The remaining `qrmi` line-number citations kept their numbers on the argument
that the pinned 0.17.2 tag was not moving. The pin moved. Those citations are
now anchored to 0.17.2 as a historical reference and have drifted against
0.24.0, which is the version this document otherwise describes. They are marked
where they appear. The lesson generalizes past this document: a citation whose
validity rests on a pin is only as stable as the decision not to upgrade.

**Status (2026-07-28):** the comparison now has live-hardware backing. Both
interfaces ran against the ORNL IQM 20-qubit system through the QFw shim:
device introspection through each returned the same normalized topology (20
qubits, 30 edges, in agreement), and one canonical circuit executed through
each returned identical counts in the same normalized result record. Axis
entries below that cite hardware observations derive from those runs.

**Resolution:** the conclusions this comparison supports are collected in the
companion document [`qrmi-qdmi-resolution.md`](./qrmi-qdmi-resolution.md).
This document records what was observed. The resolution records what should
follow from it.

## Comparison Axes

This table defines the comparison axes. It describes what each axis means and
why the behavior matters. The QRMI and QDMI behavior for each axis is recorded
in the detailed sections that follow.

| Axis | Meaning | Why it matters |
|---|---|---|
| Interface role and scope | The part of the quantum resource stack the interface is trying to own. | Establishes whether the interface targets applications, resource managers, device providers, schedulers, or multiple layers. |
| Device discovery and capability advertisement | How devices, supported calls, supported formats, and resource capabilities are exposed. | Allows software above the interface to decide which device or implementation can satisfy a request. |
| Admission | The resource-manager-facing decision path for accepting, delaying, or rejecting quantum resource requests. | Prevents uncontrolled oversubscription and gives site schedulers a device-aware admission decision. |
| Admission control configuration | The operator-facing controls used to select and tune admission policy. | Lets a site configure quality of service, allocation limits, rate limits, credit policy, and other admission behavior. |
| Device scheduler control | The operator-facing controls used to configure local device scheduling policy. | Allows a site to choose how accepted quantum tasks are ordered before they reach the QPU. |
| Runtime submission | The application-facing path for submitting quantum work for execution. | Defines the execution model, accepted payloads, async behavior, and the boundary between portable API and provider-specific job format. |
| Job lifecycle and results | The calls used to track status, cancel jobs, collect results, retrieve metadata, and inspect errors. | Keeps execution state and result handling consistent across providers and adapters. |
| Device introspection | Device identity, qubits, operations, topology, supported loci, and backend properties. | Supplies the information needed for placement, validation, compilation, scheduling, and reporting. |
| Calibration and quality data | Calibration set identity, quality metrics, fidelity data, coherence data, validity, and provenance. | Makes device quality visible to applications, schedulers, and monitoring tools. |
| Telemetry | Runtime health, load, queue state, availability, timing, and operational counters. | Supports observability and dynamic runtime policy decisions. |
| Device authentication | The mechanism used to authenticate to the quantum hardware provider. | Determines how credentials are obtained, scoped, injected, used, and revoked. |
| Control-plane authorization | The mechanism used to authorize protected service-control calls. | Separates user execution rights from privileged operations such as policy changes and credential injection. |
| Data normalization | Conversion from provider-native or interface-native payloads into common records. | Allows downstream software to consume device, calibration, telemetry, and result data without provider-specific parsing. |
| Program representation and placement | The accepted program formats, execution options, and logical-to-physical placement representation. | Determines whether applications can submit portable programs and whether placement intent survives to the provider. |
| Error model | How errors, return codes, provider failures, and retryability are represented. | Enables consistent diagnostics and prevents every caller from handling library-specific failures differently. |
| Extensibility and versioning | How the interface grows, advertises optional features, and preserves compatibility. | Allows independent API categories and provider implementations to evolve without treating the whole interface as one monolith. |

## Interface Role And Scope

Which part of the stack is each interface trying to own? The names point at
different answers — resource management against device management — and the
structure of each project bears that out more clearly than any single call
does. This axis is the frame for the rest of the document: several differences
recorded elsewhere follow from the two occupying different layers.

<details>
<summary><strong>QRMI Behavior</strong></summary>

QRMI describes itself as a "thin and vendor agnostic layer to access, control
and monitor underlying on-prem or cloud quantum computers". Its unit is a
*resource*, and the whole interface is one `QuantumResource` trait of twelve
async methods, with no separation between the API a caller uses and the API a
vendor implements. A caller uses the trait that a resource implements.

Vendor support lives inside the project. `src/` carries `ibm.rs`, `iqm.rs` and
`alice_bob.rs` with their supporting modules, so adding a backend means adding
code to QRMI and releasing QRMI. That is why `ResourceType` is a closed
enumeration of seven backends: the set of supported vendors is a property of
the QRMI build.

The vocabulary is resource-management vocabulary — `acquire` / `release`,
`ResourceProvider.resources`, a `least_busy` selector — and the deployment
story is a scheduler's. Resources are discovered from a configured resource map
or from the job environment a SLURM prologue populated, and the ecosystem
includes a SPANK plugin to do that populating. QRMI expects something above it
that has already decided which resources this job may touch.

It also ships more than one binding: a Rust core, a generated C header
(`cbindgen`), and Python bindings, which is consistent with wanting to sit
under a variety of resource-manager and middleware code.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI's unit is a *device*, and the interface is explicitly two-sided. Two
headers define two audiences:

| Header | Functions | Implemented by |
|---|---|---|
| `client.h` | 15 | the interface itself; called by tools and applications |
| `device.h` | 18 | the vendor, for each device |

The device side mirrors the client side — `QDMI_device_initialize` /
`_finalize`, `QDMI_device_session_alloc` / `_init` / `_free`, the three
`_query_*_property` calls, `QDMI_device_session_create_device_job`, and the
job operations — so a vendor implements a defined ABI rather than contributing
code to QDMI.

That is the structural consequence: devices are separate shared libraries,
loaded at runtime. In the QFw shim the IQM device library is loaded by path
through MQT Core's FoMaC loader. The set of supported devices is a property of
what is installed, not of what QDMI was built with.

The vocabulary is device vocabulary — sites, operations, loci, program formats,
typed properties — and there is no admission, reservation, scheduling or
site-policy surface anywhere in either header. MQT Core's FoMaC sits above QDMI
as a convenience layer for tools, which is the kind of consumer QDMI is shaped
for: compilers, mappers, and analysis code that need to know what a device is
and can do.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

**They are not competing for the same slot.** QRMI is a resource-manager-facing
interface: it wants to be what a scheduler or middleware layer calls to reach
any vendor's machine, and it handles vendor plurality by absorbing vendors into
its own tree. QDMI is a provider-facing contract: it defines what a device must
implement so that tools above it can be vendor-neutral, and it handles vendor
plurality by delegating to vendors behind an ABI. QRMI sits above where QDMI
sits, and in principle a QRMI backend could be implemented over QDMI.

Much of what this document records elsewhere follows from that. QRMI has
`acquire` / `release` and QDMI has no admission primitive at all, because
admission is a resource-manager concern and QDMI is not one. QDMI advertises
capability per device while QRMI implies it from a type constant, because
QDMI's whole purpose is describing devices to software that has not met them
before. QRMI hands back raw vendor payloads while QDMI returns typed neutral
values, because a resource manager is passing data through and a tool is
consuming it.

**The layering claim has since been tested from the QDMI side, and it held.**
MQT Core 3.9 added `mqt.core.qdmi.slurm.open_device_from_license()`, which reads
`SLURM_JOB_LICENSES` and opens the QDMI device whose stable ID matches the
license name. It ships with a documented cluster tutorial and a CI fixture
running a real two-node Slurm cluster. On the face of it that is QDMI reaching
into the resource manager, which is the layer this axis assigns to QRMI.

What crossed the boundary is worth being exact about, because only one thing
did. The adapter performs **device selection** and nothing else. MQT Core's own
documentation separates four controls and keeps them independent: Slurm admits
jobs and accounts for the license count, the adapter uses the license
environment to select a device, the provider reports availability and queue
state, and the provider or the operating system authorizes access. The function
docstring says plainly that it does not verify a Slurm allocation, authenticate
the caller, or authorize device access.

**And it gives a reason why it cannot, which is the sharpest statement of the
layering argument found anywhere in either project.** From the tutorial: a
lookup through a more trustworthy Slurm interface would still not make MQT Core
an access-control boundary, because a program can call
`driver.open_device(device_id)` directly. A device contract sits below the
enforcement point by construction. Anything it checks can be bypassed by calling
the layer underneath, which is the layer it *is*. So the question of whether
QDMI could grow into a resource manager is not a matter of scope or ambition. A
device interface cannot enforce admission on itself, and the moment it tried,
callers would route around it.

That is the same conclusion this axis reached from the other direction, arrived
at independently by the people implementing the device side, and it is stronger
evidence for the two-layer reading than the original argument was.

**Note also that QDMI is growing in the other direction at the same time.** 3.9
added a PennyLane device alongside the existing Qiskit backend, so the same
release reaches down toward the resource manager for device selection and up
toward application frameworks for execution. Neither move is toward QRMI's
position. A device contract acquiring adapters on both sides is what a device
contract doing well looks like.

**The plurality models have different costs, and neither is free.** QRMI's
concentrates integration effort in one project: every new backend is a change
to QRMI, which gives consistency and a single place to reason about behaviour,
at the price of the project becoming a bottleneck and of vendor code being
maintained by people who do not own the hardware. The `acquire()` divergence
recorded under Admission is a symptom — three backends in one tree with three
different meanings for one call, and nothing forcing them into agreement.
QDMI's model pushes effort to vendors and scales without a bottleneck, at the
price that conformance is only as good as the ABI is specified, and that the
caller depends on a library the project does not control. The calibration
down-select recorded under Calibration And Quality Data is a symptom of that
one: what reaches the caller is whatever the device library chose to map.

**The shim flattens two layers into one, and pays for it.** QFw's
`svc_lib_qpm` treats QRMI and QDMI as interchangeable providers of the same
calls, selected per call by a hand-maintained capability map. That map exists
because the two interfaces do not offer the same calls — which is not an
oversight in either, but a consequence of their occupying different layers. The
map is a useful experiment precisely because it makes the mismatch concrete,
but it should not be mistaken for evidence that the two are alternatives.

**For a common spec.** The first decision is which layer is being specified,
because the two are separable and both are needed. A device contract says what
a device is and can do; a resource contract says how work is admitted, placed,
accounted and reported against a fleet of them. Trying to write one interface
that does both produces exactly the tensions visible here — admission calls
that mean different things per backend, capability that is implied rather than
advertised, and telemetry that is neither the device's nor the scheduler's.

If a two-layer split is adopted, QDMI's client/device separation is worth
studying as precedent: it is the same shape as a caller API plus a provider ABI,
with the property-query model allowing new properties without breaking the ABI.
QRMI's contribution to that picture is the part QDMI does not attempt — the
scheduler-facing side, where resources are discovered from a job environment
and a site's decision is carried rather than re-made.

</details>

## Device Discovery And Capability Advertisement

Discovery is how software above the interface learns which devices exist;
capability advertisement is how it learns what each one accepts, so it can
decide which device or implementation can satisfy a request. The two interfaces
answer these from different places — one from site configuration, the other
from the device.

<details>
<summary><strong>QRMI Behavior</strong></summary>

Discovery is configuration- and scheduler-driven, not a query to the provider.
A resource is named, and the caller constructs it directly:

```python
qrmi = QuantumResource(resource_id, ResourceType.IQMServer)
```

The supporting machinery is local. `Config` (`load`, `resource_map`) reads a
resource map; `ResourceDef` describes one entry (`name`, `resource_type`,
`environment`, `is_dynamic`); `ResourceProvider` exposes `resources` over that
set, plus a `least_busy` selector. In a batch deployment the set comes from the
job environment: `get_job_qpu_resources_and_types()` reads
`QRMI_JOB_QPU_RESOURCES` / `SLURM_JOB_QPU_RESOURCES` and the matching `_TYPES`
variables, which the SPANK plugin populates — so the resource manager tells the
job what it was given, and QRMI reads that answer rather than asking the
device.

`is_accessible()` then confirms a named resource is reachable.

Capability is carried by the resource *type* rather than advertised by the
resource. `ResourceType` enumerates seven backends (`IQMServer`,
`IBMQuantumComputeService`, `IBMQiskitRuntimeService`, `IBMQuantumSystem`,
`AliceBobFelis`, `PasqalCloud`, `PasqalLocal`) and `Payload` four variants
(`IQMServer`, `QiskitPrimitive`, `AliceBobFelis`, `PasqalCloud`). Knowing the
type tells the caller which payload to build, but that mapping is knowledge the
caller must hold: it is not queryable, and there is no call that reports which
program formats or which operations a given resource accepts. Every resource
implements the same twelve-method trait, so the API surface is uniform by
construction; whether a particular call is meaningful for a particular backend
is found out by making it.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

Discovery has two forms, and the second one is new since the previous baseline.

The original form is a query against the session. `QDMI_SESSION_PROPERTY_DEVICES`
enumerates the devices a session can see, surfaced in MQT Core as
`Session.get_devices()`. Devices reach a session through the driver's device
libraries.

The second form is a registry. MQT Core 3.8 replaced the runtime loader
(`add_dynamic_device_library`, which took a library path and returned an open
device in one step) with registration by a stable device ID followed by an
explicit open. A caller registers a `DeviceDefinition` naming an ID, a library
path, and a symbol prefix. Registration validates and stores that definition
and loads no native code. `open_device(id)` then creates a fresh session.
MQT Core 3.9 added `registered_device_ids()`, which returns the enabled IDs
without loading any device library, and ships bundled device models under
stable IDs, so a fresh install already enumerates `mqt.ddsim.default`,
`mqt.sc.iqm.garnet`, `mqt.sc.iqm.emerald`, and others. QDMI-on-IQM publishes
its own ID, `iqm.default`, for callers to register the packaged library under.

The distinction matters for this axis. The old model could only tell a caller
what was reachable after it had loaded and initialized native code for each
candidate. The new one separates the catalogue from the loading, so a caller
can enumerate what a host is configured to offer, and pay the cost of opening
only the one it picks. That is closer to what a resource manager needs than
what the session enumeration alone provided.

3.9 added a third form, narrower than either. `mqt.core.qdmi.slurm`'s
`open_device_from_license()` reads `SLURM_JOB_LICENSES` and opens the device
whose stable ID matches the license name, so the site's own admission decision
becomes the selection. It is a discovery mechanism in which the caller does not
choose. What makes that possible is the stable-ID registry: a name that means
the same thing to SLURM's configuration and to the QDMI driver is what lets an
external system name a device at all. See Admission for what it does and does
not decide.

Capability is advertised by the device and queried per property.
`Device.supported_program_formats()` returns the formats that device accepts,
drawn from a defined vocabulary of fourteen (`QASM2`, `QASM3`, four `QIR_*`
variants, `QPY`, `IQM_JSON`, `CALIBRATION`, and five `CUSTOM` slots). Alongside
it are `name()`, `version()`, `library_version()`, `status()`,
`qubits_num()`, `needs_calibration()`, and the `sites()` / `operations()`
model, which states which operations are available on which loci.

The negative answer is typed as well: a property a device does not implement
returns `QDMI_ERROR_NOTSUPPORTED` rather than an empty or invented value, so
absence of a capability is distinguishable from absence of data.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

**The two answer "what is out there" from opposite directions.** QRMI takes the
answer from the site: a configured resource map, or the job environment the
scheduler populated. QDMI takes it from the session: an enumeration of the
devices its loaded device libraries expose. Neither is obviously the right
default. QRMI's model fits an HPC deployment, where the resource manager has
already decided what a job may touch and the interface should not offer more;
QDMI's fits a client that must find out what is reachable before deciding.
They are complementary rather than competing, and a common spec plausibly needs
both: an enumeration call, and the ability for a resource manager to constrain
what that enumeration returns.

**They differ more sharply on capability.** QRMI advertises nothing per
resource; capability is implied by the resource type, and the type-to-payload
mapping lives in the caller. QDMI advertises per device and per property, with
program formats as an explicit list and `QDMI_ERROR_NOTSUPPORTED` as a typed
negative. That difference is the same one visible in Runtime Submission: QDMI
declares the program format separately from the program bytes, so it has
somewhere to put a format negotiation; QRMI's payload is typed by resource, so
format compatibility is not expressible at the interface at all.

**A caller that spans both must supply the missing model itself.** The QFw shim
routes each API call to a library per resource, and it does so from a
hand-maintained capability map in its descriptor — which calls each library
serves for each device, plus a preference to break ties when both do. That map
exists because neither interface can answer "which of these calls will work for
this resource": QRMI does not advertise capability, and QDMI advertises what
the *device* supports rather than what the *interface path* supports. The map
is therefore maintained by hand and can drift from the libraries it describes —
which it did, silently, until a routing assertion failed.

**Part of what this axis asked for has since been built.** The recommendation
below was written against the session-enumeration model. MQT Core 3.8 and 3.9
then split the catalogue from the loading, which supplies most of the
enumeration half: a registry of stable IDs, readable without loading native
code, populated from configuration files a site controls. That is the scoping
the recommendation asks for, arriving from the QDMI side. What it still does
not supply is capability. An ID in the registry says a device exists and names
the library that would serve it. It does not say what that device accepts.
Answering that still means opening the device, which means loading the library
and initializing a session, which for a hardware backend means contacting the
provider.

**For a common spec.** Discovery should be a call, with the resource manager
able to scope its results, so the same API serves both the constrained batch
case and the exploratory client case. Capability advertisement should be
queryable per resource rather than implied by a type constant, should cover
accepted program formats explicitly, and should use a typed
not-supported answer so that unsupported and unavailable are distinguishable.
The registry development adds one requirement that was not visible before:
enumeration and capability should be answerable at the same cost. A catalogue
that is cheap to read but tells a caller nothing about suitability moves the
expense rather than removing it.
The test is whether a portable caller can select a device and construct a valid
request without holding out-of-band knowledge about the backend — which today
it cannot do through either interface alone.

</details>

## Admission

Admission is the resource-manager-facing decision path: accept a quantum
resource request now, delay it, or reject it. The question for each interface
is what a site scheduler can call to make that decision, and what a successful
call actually guarantees.

<details>
<summary><strong>QRMI Behavior</strong></summary>

QRMI has admission-shaped primitives. `is_accessible()` reports reachability,
and for IQM it is real: it calls the provider's health endpoint
(`get_qc_health_v1`) and returns the reported health. `acquire()` and
`release()` bracket a claim on the resource, and `ResourceProvider` offers a
`least_busy` selector over the configured set.

What `acquire()` means depends on the backend, and the caller cannot tell which
it got.

| Backend | `acquire()` |
|---|---|
| IBM Qiskit Runtime Service | real: refreshes credentials, reuses or creates a provider session |
| IBM Direct Access | `Ok(Uuid::new_v4().to_string())` — a generated id, no provider contact |
| IQM Server | `Ok(Uuid::new_v4().to_string())` — a generated id, no provider contact |

For IQM, `release()` is `Ok(())`. Both return success and neither reaches the
device. The IQM implementation's doc comment describes behaviour it does not
have — "Deletes the current session. This sends a DELETE request to
`/sessions/{session_id}/close`" — which is the Qiskit Runtime text left in
place. The Direct Access implementation is at least honest about it: "Direct
Access does not support session concept, so simply returns dummy ID for now."

There is no capability flag distinguishing the two cases. A resource manager
that calls `acquire()` receives a token whether or not anything was reserved.

Separately, QRMI ships a SLURM SPANK plugin, and that is where site admission
actually happens in a batch deployment: the scheduler decides, and the prologue
publishes the granted resources into the job environment for
`get_job_qpu_resources_and_types()` to read. Admission is the scheduler's, and
QRMI's role is to carry the result.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI has no admission primitive. The client API is fifteen functions covering
sessions, device and property queries, and the job lifecycle:

```
QDMI_session_alloc / _init / _set_parameter / _query_session_property
QDMI_device_query_device_property / _query_site_property / _query_operation_property
QDMI_device_create_job
QDMI_job_set_parameter / _submit / _check / _wait / _get_results / _query_property / _cancel
```

There is no acquire, reserve, lease, or lock. A session is a client-side
context for reaching devices, not a claim on one, and nothing in the interface
lets a caller ask to hold a device or be told to wait.

What exists that bears on admission is observational: the typed device status
(`QDMI_DEVICE_PROPERTY_STATUS`) and `needs_calibration()` let a caller decide
not to submit. Beyond that, work is submitted and the provider's own queue
decides when it runs. `QDMI_job_cancel` withdraws a job after the fact.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

**Neither interface gives a site scheduler a device-aware admission decision.**
QDMI has no admission call at all. QRMI has one whose meaning varies by
backend, and for IQM it is a no-op that returns success. Both leave the real
decision to the provider's queue, which the caller cannot see into.

**QRMI's version is the more instructive failure.** A no-op that returns a
plausible token is worse than an absent call, because it is indistinguishable
from a working one. A resource manager written against `acquire()` will behave
correctly on IBM Qiskit Runtime and silently hold nothing on IQM and IBM Direct
Access, with no error, no warning, and no capability flag to check. This is the
Device Discovery gap with consequences: capability is not advertised, so a
caller cannot ask whether the primitive it is about to rely on is implemented.
That an implementation carries another backend's documentation for behaviour it
does not have is a symptom of the same thing.

**Admission needs the telemetry neither interface supplies.** Deciding to
accept, delay, or reject requires knowing what the device is doing — queue
depth, current load, expected wait. As recorded under Telemetry, neither
interface exposes any of it, and the provider timing that would at least allow
learning from history is discarded by both. So even a correct admission call
would have nothing to decide on. The two gaps compound: no state to decide
from, and no primitive that reliably enforces a decision.

**What works today is outside the interface.** In a batch deployment the
scheduler admits, and QRMI's SPANK plugin publishes the outcome into the job
environment. That is a real, working admission path, and it is worth being
precise about why: SLURM owns the decision because it holds the state — the
queue, the allocation, the policy. Neither quantum interface holds any of that.

**The QDMI side reached the same conclusion independently, and picked a
different SLURM mechanism to do it.** MQT Core 3.9 ships a documented cluster
tutorial and a CI fixture in which admission is a **SLURM license** whose name
is a stable QDMI device ID, with `mqt.core.qdmi.slurm.open_device_from_license()`
reading `SLURM_JOB_LICENSES` to pick the device. SLURM admits and accounts, and
QDMI is handed the answer. No admission primitive was added to QDMI, which is
the point: faced with needing admission, the device-side project reached for the
resource manager rather than growing one.

There are now three site-side mechanisms visible across these projects, all
doing the same job through different SLURM features. QRMI uses a SPANK plugin
publishing into the job environment. MQT Core's example uses licenses. QFw uses
GRES, requesting `--gres=qpu:1`. Licenses are cluster-wide counters and suit a
remote QPU with no local device file. GRES is per-node and, with a `File=` entry
and `ConstrainDevices=yes`, is the only one of the three that can actually
restrict what a job opens. That three independent integrations picked three
different mechanisms, and that the choice turns on whether the device is local,
is a reasonable measure of how unsettled this is. It is also the clearest
argument in this document that admission belongs in a resource contract that
says which mechanism means what, rather than being rediscovered per site.

**For a common spec.** Three things follow. An admission primitive must have
defined semantics and be discoverable: if a resource does not support holding,
the caller must be able to learn that before relying on it, and a call that
cannot enforce should fail rather than return a token. Admission needs the
device-state telemetry to be a decision rather than a guess, which makes this
axis dependent on Telemetry rather than separable from it. And the division of
labour with the site scheduler should be stated deliberately — the resource
manager is likely to remain the admitting authority, in which case the
interface's job is to give it device state and to honour, not duplicate, its
decisions.

</details>

## Admission Control Configuration

The operator-facing controls a site uses to select and tune admission policy:
quality of service, allocation limits, rate limits, credit policy. This is the
first of three consecutive axes covering operator surfaces, and the short
answer for all three is that neither interface has one. That is not an
oversight — it follows from the roles established in Interface Role And Scope —
but it is worth recording precisely what each does have nearby, because the
adjacent machinery is easy to mistake for policy.

<details>
<summary><strong>QRMI Behavior</strong></summary>

What QRMI has is deployment configuration, not policy. `Config` loads a
resource map naming which resources exist and what type each is, and per-
resource behaviour is tuned through environment variables keyed by resource id
— `{backend}_QRMI_JOB_TIMEOUT_SECONDS`, and the credential and endpoint
variables for each backend family.

These are knobs, but they are not an operator control plane. They are read at
construction by whichever process is using the resource, and in a batch
deployment they are written into the job environment by the SPANK prologue. A
site expresses policy by controlling what the prologue writes, which is to say
by configuring SLURM, not QRMI. There is no call to set a limit, no quota or
credit concept, and no rate limiting.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI has no admission policy surface at all. Its session parameters are
identity and connection settings — `TOKEN`, `USERNAME`, `PASSWORD`, `AUTHFILE`,
`AUTHURL`, `PROJECTID` — plus five custom slots. Job parameters are
`PROGRAM`, `PROGRAMFORMAT`, `SHOTSNUM` and five custom slots. Nothing in either
family expresses a limit, a quota, a rate, or a class of service.

`QDMI_SESSION_PARAMETER_PROJECTID` is the closest thing to an allocation
concept, and it is an input the caller supplies for the provider's own
accounting rather than a control a site operator sets.

What QDMI does offer admission is state to decide on: the device status
enumeration is `IDLE`, `BUSY`, `CALIBRATION`, `MAINTENANCE`, `ERROR`,
`OFFLINE`. That is a usable operational vocabulary, and it is an input to a
decision, not a policy control.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

Neither interface lets a site operator configure admission, and the deployments
in question do configure it — in SLURM, through partitions, QoS, association
limits and reservations. The policy layer exists; it simply sits above both
interfaces and knows nothing about quantum devices beyond what a GRES count
expresses.

The gap that matters is therefore not the missing controls but the missing
inputs. A site scheduler already has the mechanisms to express quality of
service and limits; what it cannot do is make those decisions device-aware,
because as recorded under Telemetry neither interface reports queue depth,
load, or expected wait. QDMI's status enumeration is the most useful thing
either provides, and it distinguishes only six coarse states.

**For a common spec.** If the resource manager remains the admitting authority
— which Interface Role argues it should — then the interface's obligation is to
supply decision inputs rather than to grow a policy API of its own: current
load and queue depth, an expected-wait estimate, a device state vocabulary at
least as rich as QDMI's, and enough accounting identity to reconcile usage
afterwards. Adding a second policy engine below SLURM would create two
authorities with no protocol between them, which is the failure the next axis
describes.

</details>

## Device Scheduler Control

How accepted work is ordered before it reaches the QPU, and what a site can say
about that ordering. This is the axis where the absence has the clearest
operational consequence, because ordering is being decided twice.

<details>
<summary><strong>QRMI Behavior</strong></summary>

QRMI submits; it does not order. `task_start` takes a payload and returns a
task id, with no priority, deadline, weight, or queue selection anywhere in the
trait.

The one ordering-adjacent construct is IQM-specific: `Payload::IQMServer`
carries a `use_timeslot` boolean, passed through to the provider's job
submission. It selects whether the job runs inside a reserved timeslot — a
provider-side reservation concept — rather than expressing a relative ordering
among the caller's own jobs. It is also, being part of the payload, typed by
resource and unavailable to any other backend.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI has no scheduling control. The complete job parameter set is
`QDMI_JOB_PARAMETER_PROGRAM`, `_PROGRAMFORMAT`, `_SHOTSNUM`, and five `CUSTOM`
slots. There is no priority, no deadline, no ordering hint, and no queue
selection.

`QDMI_job_cancel` withdraws a submitted job, which is the only influence a
caller has over what the device does next, and it is subtractive.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

**Work is ordered twice, by two schedulers that cannot talk to each other.** A
site scheduler admits and orders jobs according to its own policy — fair share,
priority, backfill, reservations. Those jobs then submit to the device, whose
provider-side queue orders them again according to policy the site did not set
and cannot see. Neither interface carries the first decision into the second.

The consequence is not subtle. Two jobs the site deliberately ordered may reach
the device in that order and execute in the opposite one. A high-priority job
has no way to say so. A reservation the site granted means nothing at the
device. And because provider timing is discarded by both interfaces (Telemetry),
the site cannot even observe after the fact that its ordering was not honoured
— it sees total wall time and cannot separate queue from execution.

This is the sharpest example in this document of the two-layer problem the
Interface Role axis describes. Ordering is genuinely a resource-manager
concern, so it is reasonable that QDMI, a device contract, has no view on it.
But it is equally true that a resource manager cannot do its job if its
decisions are silently re-made below it, and neither interface provides the
channel that would let it.

**For a common spec.** Submission needs to carry scheduling intent — at
minimum a priority or ordering key and a deadline or expiry, with defined
behaviour when the provider cannot honour them, including saying so rather than
silently reordering. Where a site holds a reservation, submission needs to
reference it, which `use_timeslot` gestures at for one backend and one provider
concept. And the provider's own queue position and expected wait need to be
observable, so a scheduler can detect divergence between the order it chose and
the order it got. Without that channel, quantum resources cannot be scheduled
by an HPC site in any meaningful sense — they can only be submitted to.

**One of those three requirements has now been met on the QDMI side, which
usefully separates the other two.** QDMI 1.3.3 named
`QDMI_JOB_PROPERTY_QUEUEPOSITION` and `QDMI_DEVICE_PROPERTY_QUEUELENGTH`, MQT
Core 3.9 bound both, and iqm-qdmi 1.4.0 serves them. A scheduler on the QDMI
path can now ask how deep the queue is and where its job sits, and get an
answer. On the QRMI side the provider's job timeline reaches the caller through
`task_logs()`, but as formatted text rather than data (see Telemetry), so that
half is present and not yet usable.

Carrying intent **down** has not moved at all. There is still no priority, no
deadline, no ordering key, and no way for a site to tell a provider that its
reservation means something. That asymmetry is worth stating plainly, because
the two halves have different politics. Reporting queue state costs a provider
nothing and reveals little. Honouring an external ordering decision means
ceding control of the provider's own queue to a customer, which is a commercial
question before it is a technical one.

That prediction is now partly tested rather than speculative. The observability
half went from specified to shipped in a single release once there was a named
property to fill. The intent half has not moved in any release observed here.
Observability arriving alone converts the problem from invisible to merely
unfixable: a site can now measure that its ordering was discarded and still has
no way to prevent it. That is progress, and it is worth being precise that it is
the cheaper half.

</details>

## Runtime Submission

Runtime submission is the application-facing path for executing quantum work.
This axis focuses on the public API used to submit a task, the payload form that
API accepts, and the layer responsible for translating user programs into
provider jobs.

<details>
<summary><strong>QRMI Behavior</strong></summary>

The QRMI public runtime API is the `QuantumResource` task lifecycle. For an IQM
resource, the application creates a resource object, acquires it, builds an
IQM-specific payload, submits the task, polls status, retrieves results, and
releases the resource.

```python
qrmi = QuantumResource(resource_id, ResourceType.IQMServer)
lock = qrmi.acquire()

payload = Payload.IQMServer(
    iqmjson=iqm_json,
    job_type="circuit",
    use_timeslot=False,
    tag=None,
)

job_id = qrmi.task_start(payload)
status = qrmi.task_status(job_id)
result = qrmi.task_result(job_id)
logs = qrmi.task_logs(job_id)

qrmi.task_stop(job_id)
qrmi.release(lock)
```

For the IQM resource type, `Payload.IQMServer` carries an IQM JSON string. The
low-level runtime API therefore exposes a provider-specific payload shape at
the submission boundary. QRMI also has a Qiskit adapter. That adapter converts
Qiskit circuits into IQM run-request JSON and then calls the same QRMI runtime
path with `Payload.IQMServer`. The adapter improves usability for Qiskit users,
but it is layered above the public QRMI resource API.

For the IQM resource, `Payload.IQMServer.iqmjson` is the entire run request, not
a single circuit. It carries the `circuits` array, the shot count, and the
calibration set id, and is POSTed as the job body. In the QFw shim the QRMI
driver builds it as an iqm-client `RunRequest` and serializes it with
`model_dump_json()` -- the same object the QRMI Qiskit adapter produces. The
execution parameters (shots, calibration set) live inside this opaque provider
blob.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI separates the public client API from the provider-side device
implementation. The public runtime surface is the client job interface in
`HAL/QDMI/include/qdmi/client.h`. A client initializes a session, discovers
available devices, creates a `QDMI_Job`, sets job parameters, submits the job,
waits or polls for completion, and retrieves typed results.

```c
QDMI_device_create_job(device, &job);
QDMI_job_set_parameter(job, QDMI_JOB_PARAMETER_PROGRAMFORMAT,
                       sizeof(format), &format);
QDMI_job_set_parameter(job, QDMI_JOB_PARAMETER_PROGRAM,
                       program_size, program);
QDMI_job_set_parameter(job, QDMI_JOB_PARAMETER_SHOTSNUM,
                       sizeof(shots), &shots);

QDMI_job_submit(job);
QDMI_job_check(job, &status);
QDMI_job_wait(job, timeout);
QDMI_job_get_results(job, QDMI_JOB_RESULT_HIST_KEYS,
                     key_size, keys, &key_size_ret);
QDMI_job_get_results(job, QDMI_JOB_RESULT_HIST_VALUES,
                     value_size, values, &value_size_ret);
QDMI_job_free(job);
```

The job payload is described by typed parameters. The common parameters are
program format, program data, and shot count. The result path is also typed.
Common selectors include `QDMI_JOB_RESULT_SHOTS`,
`QDMI_JOB_RESULT_HIST_KEYS`, and `QDMI_JOB_RESULT_HIST_VALUES`.

The `IQM_QDMI_device_*` functions belong to the provider-side device
implementation. They mirror the client job operations, but they are the hooks
implemented by the IQM device library rather than the portable client-facing
API. In QDMI-on-IQM, `IQM_QDMI_device_job_submit()` accepts IQM JSON, QIR base
strings, and calibration programs. The implementation builds the IQM REST job
document internally. It adds the calibration set ID, shot count, execution
options, and optional qubit mapping before sending the job to IQM.

QDMI-on-IQM also exposes a Qiskit path through MQT Core. The Qiskit backend
loads the IQM QDMI device library and presents Qiskit-compatible execution.
That path is an adapter above QDMI, similar in role to the QRMI Qiskit adapter.

For `QDMI_PROGRAM_FORMAT_IQMJSON`, the `PROGRAM` parameter is a single circuit,
not a run request. `IQM_QDMI_device_job_submit_circuit()` wraps it -- it builds
`{"circuits": [program], "calibration_set_id": <session>, "shots": <SHOTSNUM>,
...}` and POSTs that. The shot count arrives as the typed
`QDMI_JOB_PARAMETER_SHOTSNUM` parameter, separate from the program bytes, and
the calibration set id comes from the initialized session. In the QFw shim the
QDMI driver serializes just the transcoded circuit and passes shots to
`submit_job(...)`; the run-request envelope is assembled by the device library.

Three changes to the submission surface since the earlier baseline, all in the
direction of saying what was previously implied.

Text and binary payloads are now distinguished. MQT Core 3.8 split `submit_job`
into a string form and an exact-bytes form, and the accessors follow, with
`Job.program` for text and `Job.program_bytes` for the submitted bytes. The
motivating case is QIR, where the `*_STRING` formats are text and the `*_MODULE`
formats are LLVM bitcode. Passing bitcode through a string API had been working
by accident of encoding. `IQM_JSON` is text and is unaffected.

Calibration runs got their own entry point. `submit_job` had rejected the
`CALIBRATION` and `BATCH_JOB` formats together, which left an implementation
able to report through `needs_calibration()` that a device needed calibrating
and no way to ask for one. `submit_calibration_job()` now exists, takes an
optional payload, and takes no shot count, since a calibration run executes no
circuit.

Batch submission was removed rather than left unimplemented. A batch job's
program is a list of job handles, not a byte payload, so it never fit the
signature. Passing `ProgramFormat.BATCH_JOB` to `submit_job` now raises. This
is worth noting as a positive example for the Error Model and Extensibility
axes: an operation that could not be expressed in the interface was withdrawn
and made to fail loudly, instead of being left as a shape a caller could
construct and then discover was meaningless.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

QRMI and QDMI both expose a task or job lifecycle, but they place the portable
boundary in different places.

QRMI presents a generic resource lifecycle. The submitted payload is typed by
resource. For IQM, that resource payload is IQM JSON. A portable application
therefore needs an adapter if it starts from Qiskit, QIR, OpenQASM, or another
higher-level representation.

QDMI presents a structured job lifecycle with standard job parameters and
standard result selectors. The payload still depends on the selected program
format. An application that submits `QDMI_PROGRAM_FORMAT_IQMJSON` remains tied
to IQM's circuit representation. An application that submits
`QDMI_PROGRAM_FORMAT_QIRBASESTRING` is using a more portable program
representation, assuming the target implementation supports that format.

The main design difference is the submission envelope. QRMI's low-level
runtime call is resource-oriented and provider-payload-oriented. QDMI's public
client API is job-oriented and declares the program format separately from the
program bytes. QDMI therefore has a clearer place to negotiate supported
formats, but provider-specific options currently flow through custom
parameters. That custom-parameter path will need stronger standardization if
the same API is expected to support placement, calibration selection, and
provider execution controls portably.

From the application developer perspective, both paths still leak backend
selection into the application. A direct QRMI application must choose the
resource-specific payload shape, such as `Payload.IQMServer`, and provide the
format expected by that backend. A direct QDMI application uses a common job
API, but it must still select the correct `QDMI_PROGRAM_FORMAT_*` enum for the
target implementation. In practice, this pushes each application toward its own
adapter layer or switch statement.

The two interfaces put the same provider format at different payload
granularities, which the QFw shim had to handle explicitly. QRMI's `iqmjson` is
the whole run request: the caller owns circuit-batching, shots, and
calibration-set selection, all packed into one opaque provider blob. QDMI's
`IQM_JSON` program is a single circuit, and the surrounding request -- circuits
array, shots, calibration set, execution options -- is assembled by the device
implementation, with shots supplied as a typed job parameter. So "the backend
takes IQM JSON" is not one thing across the two interfaces; it denotes a
run-request document in one and a bare circuit in the other.

This granularity difference is a small but direct argument for the qtask
envelope and payload split below. QDMI already separates the execution
parameters (shots as a typed parameter, format declared explicitly) from the
program bytes, which is closer to an envelope-plus-payload model. QRMI folds
those parameters into the provider payload, so a portable caller cannot set
shots or select a calibration set without editing an opaque provider document.

Running both submission paths against live hardware surfaced two further
consequences of this ownership split.

Serialization responsibility follows the envelope. Both QFw drivers consume the
same transcoded circuit object (an `iqm.pulse` `Circuit`, a Python dataclass).
The QRMI leg passes it into the iqm-client `RunRequest` model, whose pydantic
validation coerces the dataclass implicitly — the caller never explicitly
serializes. The QDMI leg submits the bare circuit and must therefore produce
the JSON itself; iqm-client's `to_json_dict()` helper is typed for plain dicts
and rejects the dataclass. The "same" provider circuit thus has two different
serialization owners, and a transcode layer shared across both interfaces has
to know which consumer it feeds.

Caller-owned assembly also couples every caller to the provider SDK's payload
surface and its stability. The QFw QRMI driver originally built the run request
with an iqm-client helper that turned out to be private (`_build_run_request`);
iqm-client 34.0.1 removed it, and submission broke with an import error before
any request was sent, while the public `RunRequest` model remained stable
throughout. On the QDMI path the equivalent assembly lives once, inside the
device implementation, so provider-SDK coupling is contained there instead of
distributed across callers. (Both breakages and their fixes: openQSE/QFw #31.)

The openQSE direction should separate the runtime envelope from the program
payload. The qtask envelope is the runtime object. It is similar in role to a
QDMI job because it carries task metadata, execution options, placement hints,
resource requirements, result routing, provenance, and the program payload. The
payload is the compiled quantum program. Its format should be an agreed
exchange format rather than a provider choice made independently by every
application.

In the common stack, applications should not need to call QRMI or QDMI directly.
They submit source-level code or framework circuits to the software stack. The
compiler and tool pipeline can use any internal IRs it needs while lowering the
program. Those internal IRs are separate from the runtime exchange format unless
one is explicitly selected as the exchange format.

```text
application source / framework circuit
  -> compiler/tool pipeline
  -> provider-neutral exchange payload
  -> openQSE qtask envelope
  -> QRMI/QDMI/runtime layer
  -> backend accepts exchange payload or rejects unsupported format
```

Direct QRMI or QDMI use should still be possible. In that mode, the caller
bypasses the higher openQSE stack but still submits one of the supported payload
classes. A portable caller uses the agreed openQSE exchange format. A caller
that intentionally needs provider-specific behavior uses a provider-native
payload and gives up portability for that call.

```text
direct application
  -> QRMI/QDMI/runtime layer
  -> payload is one of:
       openqse.exchange.v1
       provider.native
```

Precompiled and JIT-capable workflows can fit inside the same exchange model.
The exchange payload can carry metadata describing its compilation stage,
target constraints, and whether JIT or final lowering is allowed. This avoids
creating a separate precompiled format before there is evidence that one is
needed.

```text
precompiled portable payload / intermediate IR
  -> runtime selects target
  -> backend or runtime JITs/lowers to hardware-native program
  -> execution
```

The exchange model should not start by selecting one preferred circuit format.
The first step is to agree on the criteria used to select the portable exchange
format or formats. Candidate formats include QIR, OpenQASM, and other
well-defined IRs. Each candidate needs to be evaluated against the same
requirements. The format must be expressive enough for expected workloads,
including measurement, classical conditions, placement information, and
provider-neutral execution options. It must also scale as circuits grow. Large
workloads should not require transferring gigabytes of repeated text when a
compact representation, binary encoding, compression, or payload reference
would be more appropriate. The format should be easy to validate, version, and
extend. It should also allow provider-native extensions without forcing those
extensions into portable application code.

This split creates a direct dependency between the openQSE working groups. The
Resource Interface and Management working group defines how work is submitted
and routed to resources. The Compiler Infrastructure working group defines how
programs are lowered into portable exchange payloads.

| Working group | Owns | Needs from the other group |
|---|---|---|
| Resource Interface and Management | qtask envelope, runtime submission contract, payload-class negotiation, accepted/rejected format behavior, provider-native escape hatch, result and error expectations. | Exchange format definition, payload stages, validation rules, feature requirements, and compiler-visible constraints. |
| Compiler Infrastructure | Compiler/tool pipeline, internal IR choices, lowering strategy, portable exchange payload definition, JIT/lowering metadata, and exchange-format versioning. | qtask envelope schema, resource capability model, placement fields, execution-option fields, supported payload classes, and runtime error semantics. |

</details>

## Job Lifecycle And Results

Job lifecycle is the path from a submitted task to its results: status
tracking, waiting, cancellation, result retrieval, and the identity that ties a
result back to the provider-side job. The entries below record behavior
observed when both interfaces executed the same circuit on live hardware (the
ORNL IQM 20-qubit system) through the QFw shim.

<details>
<summary><strong>QRMI Behavior</strong></summary>

The lifecycle is the task half of `QuantumResource`: `task_start` returns the
task identifier, `task_status` is polled until terminal, `task_result` returns
the result payload, `task_logs` returns provider logs, and `task_stop` cancels.
There is no wait call; pacing is the caller's polling loop.

For an IQM resource, the identifier returned by `task_start` is the IQM
server's job UUID itself, and `task_result` is raw IQM measurement JSON. The
provider-side job identity is therefore native to the lifecycle: the caller can
correlate its task with the IQM job record afterward without any extra call.

Because the caller also assembled the run request (see Runtime Submission),
everything about the submission is available for provenance. The QFw shim's
normalized result record from the QRMI leg carries the job UUID, the provider
status string, the calibration set id, and the full run request that produced
the counts under `extensions["iqm.v1"]`.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

The lifecycle is the client job API: `QDMI_job_submit`, `QDMI_job_check`,
`QDMI_job_wait(timeout)`, `QDMI_job_cancel`, `QDMI_job_get_results` with typed
result selectors, `QDMI_job_query_property`, and `QDMI_job_free`. Compared with
QRMI it adds a blocking wait and an explicit free, and results are typed
selections (shot count, histogram keys, histogram values) rather than one raw
payload.

Through MQT Core's FoMaC Python surface the QFw shim drives this as
`submit_job(...)` returning a `Job`, then `Job.check()` until terminal, then
`Job.get_counts()`. The counts arrive typed and correct.

Job identity is available. QDMI defines `QDMI_JOB_PROPERTY_ID`, QDMI-on-IQM
answers it with the IQM job id (`QDMI-on-IQM/src/iqm_device.cpp`,
`QDMI_DEVICE_JOB_PROPERTY_ID` -> `job->job_id_`), and FoMaC exposes it as the
`Job.id` property. Provider-job correlation is therefore supported on this
path.

Calibration identity is reachable, with a caveat, and this is a correction to
the earlier baseline. No **job** property returns the calibration set a job ran
under. The **device** does expose it, in the `CUSTOM1` slot, and MQT Core 3.8's
typed custom-property query made it readable from Python. See Device
Introspection for the full correction. The caveat matters for this axis
specifically: the value read from the device is the set that is active *now*,
not the set a given job ran under. For a job that completes while a calibration
rotates, those are different, and nothing in the job record pins which one
applied. Correlating a result with its calibration therefore still depends on
the caller reading the device property close enough in time to the run, which is
a convention rather than a guarantee.

The run-request envelope was assembled inside the device implementation (see
Runtime Submission), so the caller never held the submission document either,
and no provenance of what was actually submitted is retrievable from the job.

Job retrieval was added in MQT Core 3.9: a job can now be recovered by its
provider ID through `Device.retrieve_job_by_id()`. Before that a `Job` existed
only as the object returned by submission, so a process that submitted and then
exited had no way back to the work it started. For a batch system where
submission and result collection are different processes, and possibly different
SLURM steps, that gap was structural rather than cosmetic.

Timing is absent. The job object exposes no submission or execution timestamps.
Queue position is a partial exception as of QDMI 1.3.3, which named the property
that QDMI-on-IQM 1.3.0 does not yet serve. See Telemetry.

</details>

<details>
<summary><strong>Note On An Earlier Reading Of This Axis</strong></summary>

An earlier revision of this section reported that no accessor returned the
provider job id, based on QDMI-path result records that carried only
`job: {status: completed}`. That was a defect in the QFw shim, not an interface
limitation: the driver called the property as a method (`job.id()`), and a
bare `except` swallowed the resulting `TypeError` (fixed in openQSE/QFw #33).

The episode is itself relevant to the Error Model axis. A silent fallback made
a caller bug look like a missing interface capability, and it survived a live
hardware run because the absent field was indistinguishable from a field the
provider does not supply. Interface comparisons drawn from observed payloads
need the negative results checked against the interface definition before they
are treated as interface findings.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

On lifecycle mechanics the two interfaces are equivalent for the simple case,
and the hardware run demonstrates it: the same canonical circuit submitted
through each returned identical counts in the same normalized record
(`qhw-result-v1`). The observed field-level difference is entirely in identity
and provenance:

| Normalized-record field | QRMI leg | QDMI leg |
|---|---|---|
| Field | QRMI | QDMI | Difference is |
|---|---|---|---|
| `result.counts` | `{'1': 10}` | `{'1': 10}` | — |
| job identity | task id is the IQM job id | `QDMI_JOB_PROPERTY_ID` / `Job.id` | none; both supply it |
| provider status | `completed` | job status enum | none |
| calibration identity | in the target document | not exposed | interface |
| submitted-request provenance | caller holds the run request | assembled internally, not returned | interface |
| provider timing | none through `task_result` | no job timestamps | neither supplies it |

Both interfaces supply job identity, so provider-job correlation is available
on either path. The real asymmetry is narrower than the payloads first suggest,
and it follows from where the run request is assembled. The layer that builds
the envelope is the layer that holds the calibration selection and the
execution options. QRMI leaves assembly with the caller, so the caller retains
the submitted document. QDMI-on-IQM assembles it inside the device library and
returns no record of what was submitted.

The practical consequence is provenance rather than correlation. A site can
always tie its run to a provider job id. What it cannot reconstruct from the
QDMI path is what was actually submitted and against which calibration set —
which is what usage reconciliation and incident forensics on a shared
instrument need beyond the id itself.

Neither interface reports provider-side timing on the paths exercised here.
Queue time and execution time are not separable from either, so a caller can
measure only total wall time around its own calls. That limits any comparative
latency work to end-to-end numbers and is a gap for both.

For a common spec: job identity is already common ground and should be
standardized as such. The parts that need definition are a provenance record —
what was submitted, with which calibration selection and options, retrievable
from the job regardless of which layer assembled the envelope — and a timing
model that distinguishes queue from execution.

</details>

## Device Introspection

Device introspection is the information used for placement, validation,
compilation, scheduling, and reporting: device identity, qubits, operations,
topology, supported loci, and backend properties. This axis focuses on the shape
in which each interface exposes that information.

<details>
<summary><strong>QRMI Behavior</strong></summary>

QRMI exposes introspection through `QuantumResource.target()` (and
`metadata()`). For an IQM resource, `target()` returns a single JSON document
assembled from three IQM Server REST calls: the dynamic quantum architecture
(qubits, gates, and their loci), the calibration set, and the quality metric
set. The document is raw IQM data — the field names and structure are
provider-specific, and the consumer parses the IQM shapes directly.

**This changed at QRMI 0.22.0.** The earlier baseline recorded that the
assembled document contained exactly those three components and no static
architecture, so the only qubit set `target()` reported was the dynamic,
currently calibrated one, and a consumer wanting the chip's static topology had
to go elsewhere. 0.22.0 added a fourth component,
`static_quantum_architecture`, taken from the quantum-computer-level
`static-quantum-architectures` artifact rather than from anything bound to a
calibration set. On the ORNL q20 it carries four fields: the full 20-qubit set,
the connectivity, an empty computational-resonator list, and the DUT label
`M194_F0W1388_P08_Q12`, which is the chip identity none of the other three
components supply.

It is best-effort rather than guaranteed. QRMI's own implementation notes that
the available artifacts depend on the quantum computer's Station Control
version, so a 404 for this one specifically is normal and is represented as
`null`, while the other three components are required and a failure in any of
them fails the whole call. A consumer therefore has to treat the static
architecture as optional in a way it does not have to treat the dynamic one.

The shape is worth recording because it caused a real break. The field is a
**list** of architectures, one per DUT, not a single object, and consumers
written against the previously-absent field assumed an object. In QFw the
normalizer called `.get()` on what it was handed and the first introspection
call after the upgrade failed with `AttributeError: 'list' object has no
attribute 'get'`. This is the Data Normalization hazard in its purest form: an
optional field that has never been populated is untested code on the consumer
side, and the provider filling it in is indistinguishable, from the consumer's
point of view, from a breaking change.

`target()` is not reservation-bound; it does not require `acquire()` and reads
the current data with only a valid endpoint and token. There is no separate
typed topology or property API distinct from this document; discovery and
introspection are the parsing of the raw payload.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI exposes introspection as a typed, vendor-neutral query interface. Through
MQT Core's FoMaC layer, bound in Python as `mqt.core.qdmi` since MQT Core 3.9
(the former `mqt.core.fomac` module remains as a deprecated alias through the
v3 series), a `Device` provides typed accessors:
`name()`, `version()`, `status()`, `qubits_num()`, `supported_program_formats()`,
`needs_calibration()`, `duration_unit()`, `coupling_map()`, `sites()`, and
`operations()`. A `Site` exposes `name()` (the device's real qubit label, e.g.
`"QB1"`), `index()`, `t1()`, `t2()`, and coordinates. An `Operation` exposes
`name()`, its loci through `sites()` / `site_pairs()`, `fidelity()`,
`duration()`, and `idling_fidelity()`.

The values are the device's real labels and metrics, not a provider-specific
document. Every query is session-based: the session must be initialized before
any property can be read.

MQT Core 3.8 added a typed query for the vendor custom slots,
`query_custom_property(property, value_type)`, which returns the value coerced
to the requested Python type. This is what makes the numbered `CUSTOM` slots
reachable from Python at all. Before it, the binding exposed each accessor
explicitly and had no generic property call, so anything a vendor put in a
custom slot was visible to a C++ caller and invisible to a Python one.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

QRMI presents introspection as one raw provider document behind a single call;
QDMI presents it as a set of typed, vendor-neutral properties. The QRMI form
preserves everything the provider sends — including calibration and quality
identity — at the cost of provider-specific parsing. The QDMI form is portable
and directly consumable, but exposes what its neutral property set covers.

**The calibration-set identity finding is corrected.** The earlier baseline
recorded that the provider's calibration-set identity was not reachable through
the QDMI path. That was wrong, and it is worth being precise about how, because
the mistake is instructive. QDMI-on-IQM does publish the active calibration set
ID, in the device's `CUSTOM1` slot, and has since at least 1.2.0. The source
carries an explicit comment saying so. What was missing was purely the Python
side: MQT Core bound each accessor by hand and had no generic property query, so
a value sitting in a custom slot could not be read from Python. MQT Core 3.8
added `query_custom_property` and the value became reachable.

Measured on the ORNL q20 at the current versions, the QDMI path returns
`05ce3ca0-72af-4a1b-acfb-99233fc35e9a`, byte-identical to the
`calibration_set_id` QRMI reports inside `dynamic_quantum_architecture`. The two
interfaces agree on calibration identity as well as on topology.

The corrected finding is narrower and more useful than the one it replaces. It
is not that the neutral model cannot carry provider identity. It is that the
neutral model carries it in a **numbered, unnamed slot**, so a portable caller
cannot read it. Reading `CUSTOM1` requires knowing that this vendor put the
calibration set ID there, which is out-of-band knowledge of exactly the kind a
neutral interface exists to remove. A second vendor may use `CUSTOM1` for
something else and be equally conformant. See Extensibility And Versioning,
where this is the concrete case the custom-slot argument was predicting.

A concrete consequence in QFw: `get_device_info` and `get_coupling_graph` are
served by both interfaces, but `get_backend_info` and `get_dynamic_backend_info`
are served only by QRMI, because their shape is the raw IQM architecture
document rather than a normalized record. That split is now about payload shape
alone. The two pieces of content originally cited as QDMI-side gaps have both
closed from opposite directions: QRMI gained the static architecture at 0.22.0,
and QDMI's calibration-set id became readable at MQT Core 3.8.

For a common spec: a typed, neutral property model (QDMI-like) is the more
portable basis, provided it gives provider identity and provenance a **named**
place. The calibration set ID is the test case. It is not vendor-specific in
any meaningful sense, every provider that calibrates has one, and it is the
single field that makes a measured value interpretable later. A model that
leaves it to a numbered vendor slot has not specified it.

</details>

## Calibration And Quality Data

This axis compares how each interface exposes calibration-set identity, dynamic
architecture, quality metrics, observation provenance, and normalized
calibration records. The focus is the data path from the provider call to the
object available to application or analysis code.

<details>
<summary><strong>QRMI Behavior</strong></summary>

QRMI carries the provider-native IQM calibration payload through the interface
boundary. The QFw shim receives complete IQM endpoint responses and then lets
`qhw-iqm` build a normalized record from those responses.

**QFw Entry Point**

- The QRMI shim driver calls `self._qpu().target().value`, parses the returned
  JSON, and caches it for the driver instance
  (`QFw/services/svc_lib_qpm/drivers/qrmi_driver.py`, `_target()`).
- `get_calibration_snapshot()` maps the QRMI target fields into the shape
  expected by `qhw_iqm.normalize_calibration`
  (`QFw/services/svc_lib_qpm/drivers/qrmi_driver.py`, `get_calibration_snapshot()`).
- The mapping is direct: QRMI `dynamic_quantum_architecture` becomes
  `dynamic_architecture`, QRMI `calibration_set` remains `calibration_set`,
  and QRMI `quality_metrics` becomes `quality_metric_set`.

**QRMI Target Construction**

- The QRMI Python binding maps
  `QuantumResource(..., ResourceType.IQMServer)` to the Rust `IQMServer`
  implementation (`qrmi/src/pyext.rs:102-107`).
- Python `.target()` calls `self.qrmi.target().await`
  (`qrmi/src/pyext.rs:226-231`).
- The Rust `IQMServer::target()` method builds one JSON object from three IQM
  calibration-set calls:

| QRMI target field | IQM client call | Code |
|---|---|---|
| `dynamic_quantum_architecture` | `get_dynamic_quantum_architecture_v1` | `qrmi/src/iqm/server.rs:236-251` |
| `calibration_set` | `get_calibration_set_v1` | `qrmi/src/iqm/server.rs:253-268` |
| `quality_metrics` | `get_quality_metrics_v1` | `qrmi/src/iqm/server.rs:270-285` |

The generated IQM client sends those calls to the direct calibration-set
endpoints:

| Endpoint | Code |
|---|---|
| `/api/v1/calibration-sets/{qc}/{cal_set}` | `qrmi/dependencies/iqm_client/src/apis/calibration_sets_api.rs:66-71` |
| `/api/v1/calibration-sets/{qc}/{cal_set}/dynamic-quantum-architecture` | `qrmi/dependencies/iqm_client/src/apis/calibration_sets_api.rs:112-117` |
| `/api/v1/calibration-sets/{qc}/{cal_set}/metrics` | `qrmi/dependencies/iqm_client/src/apis/calibration_sets_api.rs:159-164` |

**Normalized Output**

- `qhw-iqm` reads the full `calibration_set` and `quality_metric_set` objects
  (`qhw-iqm/src/qhw_iqm/normalize.py`, `normalize_calibration()`).
- `normalize_calibration()` counts both observation arrays and stores the IQM
  observation sets under `extensions["iqm.v1"]`
  (`qhw-iqm/src/qhw_iqm/normalize.py`, `normalize_calibration()`).
- `_iqm_observation_set()` preserves the observation-set identity fields and the
  full `observations` arrays (`qhw-iqm/src/qhw_iqm/normalize.py`, `_iqm_observation_set()`).

The QRMI raw target contains the full provider endpoint responses. The
qhw-normalized calibration record still contains the full calibration and
quality observation arrays under the IQM extension.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI-on-IQM fetches IQM quality metrics, but it exposes a provider-neutral
projection instead of the full IQM observation sets. The raw IQM observations
are reduced inside the C++ device implementation before Python or QFw can read
them through FoMaC.

**QFw Entry Point**

- QFw registers the IQM QDMI shared library under its stable device ID and
  opens it (`QFw/services/svc_lib_qpm/drivers/qdmi_driver.py`, `_device()`).
  This replaced a one-step runtime loader in MQT Core 3.8. See Device Discovery.
- `get_calibration_snapshot()` calls
  `fomac_normalize.extract_calibration(self._device())`
  (`QFw/services/svc_lib_qpm/drivers/qdmi_driver.py`, `get_calibration_snapshot()`).
- The QFw extractor can only ask the FoMaC/QDMI object for standardized
  accessors: per-site `t1()` and `t2()`, per-operation `fidelity()` and
  `duration()`, and the device duration unit
  (`QFw/services/svc_lib_qpm/drivers/fomac_normalize.py`,
  `extract_calibration()` and its locus helpers).
- The normalized record stores only `qubit_metrics`, `gate_metrics`, and
  `duration_unit` under `extensions["qdmi.fomac.v1"]`
  (`QFw/services/svc_lib_qpm/drivers/fomac_normalize.py`,
  `to_calibration_record()`).

**QDMI-on-IQM Fetch And Down-Select**

- The IQM device session stores site identity, `t1_`, `t2_`, and operation
  fidelity maps (`QDMI-on-IQM/src/iqm_device.cpp`, the session's `sites_map_` and the
  per-site `t1_` / `t2_` members).
- QDMI-on-IQM fetches the IQM quality-metrics endpoint
  (`QDMI-on-IQM/src/internal/iqm_api_config.cpp`,
  `GET_CALIBRATION_SET_QUALITY_METRICS`, fetched by
  `Process_calibration_metrics()`).
- `Process_calibration_metrics()` flattens valid observations into a
  `dut_field -> value` map (`QDMI-on-IQM/src/iqm_device.cpp`,
  `Process_calibration_metrics()`).
- The implementation then keeps only the fields it recognizes:

| IQM `dut_field` pattern | Stored QDMI-on-IQM value | Code |
|---|---|---|
| `characterization.model.<qubit>.t1_time` | `site->t1_` | `Process_calibration_metrics()` |
| `characterization.model.<qubit>.t2_time` | `site->t2_` | `Process_calibration_metrics()` |
| `metrics.ssro.measure.<impl>.<qubit>.fidelity` | single-qubit fidelity map | `Process_calibration_metrics()` |
| `metrics.rb.prx.<impl>.<qubit>.fidelity:par=d2` | single-qubit fidelity map | `Process_calibration_metrics()` |
| `metrics.irb.cz.<impl>.<q1>__<q2>.fidelity:par=d2` | two-qubit fidelity map | `Process_calibration_metrics()` |

**Exposed QDMI Properties**

- Site queries expose index, name, T1, and T2
  (`QDMI-on-IQM/src/iqm_device.cpp`,
  `IQM_QDMI_device_session_query_site_property()`).
- Operation queries expose name, qubit count, parameter count, supported sites,
  and fidelity when a mapped fidelity exists
  (`QDMI-on-IQM/src/iqm_device.cpp`,
  `IQM_QDMI_device_session_query_operation_property()`).
- QDMI defines an operation duration property, and QFw asks for it through
  FoMaC. QDMI-on-IQM documents that IQM operation durations are not exposed by
  this provider implementation (`QDMI-on-IQM/docs/usage.md`).

The QDMI path therefore provides portable calibration properties. It does not
provide the raw IQM calibration or quality-metric observation sets to Python.

It does provide calibration **identity**, which the earlier baseline recorded as
missing. The active calibration set ID is exposed in the device's `CUSTOM1` slot
and has been since at least 1.2.0. What changed is the Python side: MQT Core 3.8
added a typed custom-property query, and the value became readable. Measured on
the q20, it matches the `calibration_set_id` QRMI reports. So the split on this
axis is between identity and bulk. Both interfaces now say *which* calibration
is active. Only QRMI hands over the observation sets that calibration contains.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

QRMI and QDMI expose different levels of calibration detail.

QRMI behaves as a provider-native data transport for IQM target data. Downstream
code receives the dynamic architecture, calibration set, and complete
quality-metric observation arrays. The qhw-normalized record retains those IQM
observation sets under an IQM-specific extension, so detailed analysis can still
inspect the provider payload.

QDMI-on-IQM behaves as a standardized property projection. It fetches the IQM
quality-metrics response, but it only exposes fields that the provider
implementation maps into QDMI properties. In the QFw/QDMI path, Python receives
T1/T2 values and selected measure/prx/cz fidelities. It does not receive the
original IQM observation objects.

Detailed calibration-analysis workloads that require the complete IQM
observation set are not supported through the QDMI path described here. Examples
include analyses over unmapped `dut_field` entries, readout error components
such as `error_0_to_1` and `error_1_to_0`, `t2_echo_time`, observation IDs,
timestamps, units, uncertainty, invalid flags, or other provider-native
observation fields. Those fields are removed by QDMI-on-IQM's down-select before
the QFw FoMaC adapter builds its `qdmi.fomac.v1` record.

The practical split is straightforward. The QDMI path described here is
suitable for portable device-property queries and simple quality summaries
based on mapped metrics. Workflows that need full IQM calibration analytics
must use the QRMI path, the native IQM path, or a QDMI extension that exposes
raw calibration and quality-metric observation sets.

The identity correction narrows what a specification has to argue about here.
Carrying the full provider observation set through a neutral interface is a real
design question with a defensible answer either way, since the payload is large,
provider-shaped, and useful to few callers. Carrying the identifier of the
calibration a measurement came from is not that kind of question. It is one
string, every calibrating provider has it, and without it the metrics on the
portable path cannot be compared across time or matched to a result. Today both
libraries supply it, one in a raw document and one in an unnamed vendor slot,
and neither in a place a portable caller can rely on by name.

</details>

## Telemetry

Telemetry is runtime health, load, queue state, availability, timing, and
operational counters — what software above the interface can observe about a
device while it is running, and what that observation costs. The cost half
matters as much as the content: a runtime that cannot afford to look cannot use
what is exposed.

The figures below are measured, not read from source. Method and raw data are
in `openQSE/QFw`, `examples/measure_shim_introspection.py` and
`examples/measurements/`.

<details>
<summary><strong>QRMI Behavior</strong></summary>

The observability surface on `QuantumResource` is `is_accessible()` for
reachability, `metadata()`, and `target()`, whose payload carries the IQM
quality-metric set alongside the architecture and calibration data. Job-level
state is `task_status()`, with `task_logs()` for provider logs.

There is no queue-state call. In the QFw result record the metadata block
carries a `queue_position` field, which the IQM path leaves null. `task_result`
returns measurement JSON and no timestamps.

**One correction to the earlier reading of this axis.** It concluded from
`task_result` that no provider-side timing reaches the caller. That is true of
`task_result` and false of the interface. `task_logs()` fetches the IQM job with
its timeline included and returns the provider's per-event record, each entry
carrying a timestamp, a source, and a status. The provider's own account of when
the job moved between states does reach the caller.

It arrives as a preformatted string. The implementation walks the timeline and
writes lines into a `String` with fixed-width padding, then does the same for
the job's messages, and returns the assembled text. A scheduler wanting to know
how much of a job's elapsed time was queueing has the data, and has to recover
it by parsing prose that QRMI generated from structured input it already held.

That is a different failure from the one the axis originally recorded, and a
more tractable one. The data is not missing. It has been flattened into a
display format at the interface boundary. This behavior predates the current
version and was present at 0.17.2, so it is a gap in the earlier reading rather
than a change in the library.

Cost of observing. `target()` issues three IQM REST calls and the shim driver
caches the parsed document per driver instance, so the cost is paid once per
instance. Measured against the ORNL 20-qubit device from outside the site:
1334.6 ms median to open and complete a first introspection (5 samples,
1238-1500 ms), then 11-18 ms for repeat calls with no network involved. The
three REST calls travel over a **single pooled TCP connection** — one TLS
handshake — because the Rust client reuses connections.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

The device carries a typed status property (`QDMI_DEVICE_PROPERTY_STATUS`,
surfaced by FoMaC as `Device.status()`), plus `needs_calibration()` and the
per-site and per-operation quality values described under Calibration And
Quality Data. Job state is the status enum from `Job.check()`.

There is no queue-state property, and the job object exposes no submission,
queue, or execution timestamps — so, as on the QRMI side, queue time and
execution time are not separable by the caller.

Queue position is a narrower case, and the more instructive one. The IQM job
submission response carries a `queue_position` field, and QDMI-on-IQM does read
it. In `IQM_QDMI_device_job_submit_circuit()` (and its calibration counterpart)
the value is parsed and appended to an INFO log message. It is not stored on the
job and it is not exposed as a property. The device job properties that
implementation serves are `ID`, `PROGRAMFORMAT`, `PROGRAM`, `SHOTSNUM`, and five
`CUSTOM` slots. So the provider supplies the caller's position in the queue, the
device library prints it, and no caller above the interface can retrieve it.

Two qualifications. The parse is guarded by a `contains("queue_position")`
check, so the field is optional from the implementation's point of view, and it
has not yet been confirmed populated on the ORNL device. And the value is a
one-time report at submission rather than a queryable property, so even if it
were exposed it would not by itself answer how deep the queue is now.

**Half of that has since been fixed, and the half that has not is now precisely
located.** QDMI 1.3.3 added two named properties for exactly this:
`QDMI_JOB_PROPERTY_QUEUEPOSITION` and `QDMI_DEVICE_PROPERTY_QUEUELENGTH`.
Neither existed in 1.3.2. MQT Core 3.9 binds both, as `Job.queue_position` and
`Device.queue_length`, each returning an optional value. So the specification
now has a defined place to put the number, and the client layer can carry it.

QDMI-on-IQM 1.3.0 still did not. Its job property handler served `ID`,
`PROGRAMFORMAT`, `PROGRAM`, `SHOTSNUM`, and the five `CUSTOM` slots, then
returned not-supported, and the submission response was still parsed only into a
log line.

**iqm-qdmi 1.4.0 closed it.** The job now stores the position and the handler
serves `QDMI_DEVICE_JOB_PROPERTY_QUEUEPOSITION`, and the same release moved the
library to QDMI 1.3.3 so the device-level `QDMI_DEVICE_PROPERTY_QUEUELENGTH`
answers too. Both were verified on the ORNL q20: `Device.queue_length()` returns
`0` where it previously raised, and `Job.queue_position` returns cleanly. It
reads `None` on that device today, which is the honest answer rather than a
failure, because a queue of length zero has no position to report.

So this finding is closed, and the sequence it took is the part worth keeping.
The value was arriving from the provider the entire time. It went from parsed
and discarded, to named in the specification but unreachable because the device
library predated the name, to reachable. Three releases across two projects to
surface an integer that had been in the submission response all along. **The
gap was never that the information did not exist. It was that no layer had been
given a defined place to put it.**

Cost of observing. The IQM device library fetches during session init, so the
cost is paid when the device is opened; property queries afterwards are local
reads. Measured on the same device and path: 3376.8 ms median cold (5 samples,
3193-3397 ms), then 12-17 ms for repeat queries, no network. Session init opened
**five separate TCP connections** — five TLS handshakes. At the measured version
QDMI-on-IQM issued each request through cpr's free-function API (`cpr::Get` /
`cpr::Post` in `src/internal/http_client.cpp`), which constructs and destroys a
session, and with it the underlying libcurl handle and its connection cache, per
call. No session was retained across requests, so each one reconnected.

**Those figures are 1.2.0-era. The mechanism moved twice since, and the path
has been re-measured at 1.4.0.** 1.3.0 replaced the free functions with a
per-request `cpr::Session`, which changed the shape of the code without changing
the outcome, since destroying the session still discarded the connection cache.
1.4.0 added a `cpr::ConnectionPool` owned by each device session and shared
across initialization, architecture refreshes, submission, polling, result
retrieval, cancellation and retries.

Re-measured on the same device and the same home broadband path the original
figures used, 5 cold samples each in a fresh process, with QRMI measured in the
same session as a control:

| | 1.2.0 / qrmi 0.17.2 | 1.4.0 / qrmi 0.24.0 |
|---|---|---|
| QDMI cold total (median) | 3376.8 ms | **1526.8 ms** |
| QDMI connections at session init | 5 | **1** |
| QDMI warm property queries | 12-17 ms | 12-16 ms |
| QRMI cold total (median), control | 1334.6 ms | 1452.0 ms |

**QDMI-on-IQM's cold cost fell by 55%, and the connection count is why.** Five
TLS handshakes became one. The same five requests are still issued and the warm
figures are unmoved, which is what isolates the handshakes as the cause.

**The comparative claim this axis made no longer holds, and that is the more
important result.** Cold start was the sharpest quantitative difference between
the two libraries, with QDMI-on-IQM costing 2.5 times what QRMI did. It is now
1526.8 ms against 1452.0 ms, a difference of about 5%, inside the run-to-run
spread. **On this measure the two libraries have converged.**

The control also answers a question the version bump raised. QRMI's `target()`
gained a fourth REST call in 0.22.0, the static architecture, and its cold cost
moved from 1334.6 ms to 1452.0 ms. The added fetch costs roughly 120 ms, and
nothing else regressed across seven releases.

An earlier attempt at this re-measurement was taken over a mobile tether and is
not reported here, because the link was not the one the baseline used. Why it
was discarded is worth recording. QRMI, the unchanged control, came in at
6265.6 ms against its documented 1334.6 ms, and a five-fold move in a library
that had not changed is what identified the measurement rather than the software
as the variable. Without a control in the same session, the QDMI figure from
that run would have looked like a plausible result.

**The payload difference is a property of the data and survives both links.**
Timing the individual REST calls, the IQM calibration-set document is
**1,264,529 bytes** against 3 KB for the dynamic architecture, and that size is
byte-identical across both measurements. The transfer time is not: 0.62 to 0.93
seconds on broadband against 7.7 to 12.4 seconds over the tether. QRMI's
`target()` fetches that document. QDMI-on-IQM does not, because it reduces the
quality metrics device-side before anything crosses the interface.

So the thick and thin split described under Calibration And Quality Data is not
only a difference in what a caller receives. It is a difference in what crosses
the wire, and how much that costs is a question about the link rather than about
either library. On broadband the 1.2 MB fetch runs about 0.2 seconds slower than
the 3 KB one and is close to free. Over the tether it cost seven to eleven
seconds more. The same implementation choice is invisible on a fast path and
dominant on a slow one, which is the same shape as the connection-handling
finding above and the reason neither should be filed as a micro-optimization.

Every figure in this axis was taken from outside ORNL, over the tunnel. The
on-site re-run remains outstanding and would compress these differences further
still.

**This one has a documented cause, and it is us.** The upstream pull request
that added the pool states its motivation as the connection behavior measured in
`openQSE/QFw#34`, the QFw measurement scripts, and reproduces the finding: five
sequential requests at initialization, each establishing its own TCP and TLS
connection because an independent session was created and destroyed per request.

That is worth separating from the other upstream fix this document records. The
QRMI null-substitution repair under Error Model is **not** claimed as caused by
this work, because there is no record of it having been reported and timing is
not evidence. Here the causal link is not inferred from timing. It is written
into the change itself.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

These observations are measured against three paths, not two. QFw can reach the
same device through QRMI, through QDMI, and through its own native service
client (`svc_iqm_qpm`) calling `iqm-client` directly. The native arm is a
control: it shows what each interface layer costs relative to not having one,
and — more usefully here — what the provider makes available before an
interface decides what to keep.

**What is exposed.** Both interfaces report device-level state — reachability
and quality data on the QRMI side, a typed device status and calibration
signals on the QDMI side — and both report job state. Neither exposes queue
depth or device load to a caller in usable form. A scheduler still cannot ask
either interface how busy the device is.

**The provider timestamp finding has to be restated, and it gets more useful in
the restating.** The earlier version said neither interface exposes any provider
timestamp. Rechecking found that both of them receive the data and lose it in
different ways, which is a better finding than absence.

| | What the provider sends | Where it stops |
|---|---|---|
| QRMI | Job timeline, one entry per state transition with timestamp, source, and status | Reaches the caller, but flattened by `task_logs()` into a preformatted display string |
| QDMI-on-IQM | Queue position in the submission response | Was parsed into an INFO log line and never stored. **Served as a job property from iqm-qdmi 1.4.0** |

Neither loss was architectural, and one of them has since been repaired. QRMI
still holds the timeline as structured data and chooses to render it.
QDMI-on-IQM parsed the integer and chose to log it until 1.4.0, which stores and
serves it. In both cases the value crossed the network, entered the library, and
was discarded above the wire and below the caller. That one of the two was fixed
by a single release, once the specification had somewhere to put the value, is
evidence for how shallow the remaining loss is on the other side.

That divides the remaining work in a way this axis did not previously make
explicit. Pushing scheduling intent **down** to the provider needs the provider
to accept it, which needs vendor cooperation and probably a specification.
Getting device and job state back **up** mostly does not. The data is already
arriving. It is being thrown away by code on this side of the boundary, and
for one of the two cases the place to put it now exists.

**That gap is not the provider's.** The native client reads the IQM job
timeline and reports the phases directly: queue wait, validation, compilation,
execution, post-processing, and a server-side total. A measured run gave 34.9
ms of queue wait and 107.3 ms of execution on a circuit whose client-side
elapsed time was several seconds. The same device, through either interface,
reports none of it.

Queue position is the same story in miniature, and it is worth stating
separately because it is not a case of an interface failing to ask. The
provider volunteers the value in the submission response. QDMI-on-IQM receives
it, formats it into a log line, and keeps no record of it on the job. The
information travels all the way to the library and stops there.

So queue-versus-execution is not information that has to be invented for a
common spec, nor obtained by instrumenting callers. It exists at the provider
and is discarded in the layer above. QRMI's `task_result` carries measurement
JSON and drops the timeline that accompanied it; QDMI's job object exposes no
timestamp property at all. That is a stronger and more actionable finding than
a symmetric absence: the requirement is to preserve what the provider already
sends, not to construct something new.

It also divides the work. Carrying scheduling intent *down* to the provider
depends on the provider accepting it, so priority, deadlines, and portable
reservation references need vendor cooperation. Getting device and job state
back *up* mostly does not, because the data is already arriving and is being
dropped by the layer above. The second half of that division is available to
implementers today.

It also bounds what the interfaces can support. Any scheduler decision that
depends on distinguishing a busy device from a slow interface — backpressure,
capacity planning, SLA accounting, deciding whether a queue is worth waiting
in — is unavailable through either interface today and available natively.

**What observation costs.** Both interfaces make repeat observation nearly
free — QRMI by caching the target document per driver instance, QDMI by serving
property queries from an initialized session. Measured repeat cost is 11-18 ms
on both, involves no network on either, and is dominated by record
normalization rather than by the interface. Neither has a warm-path advantage
over the other.

Both have a large one over the native client, which caches nothing. Every
introspection call on the native path re-fetches: 512.6 ms for a repeat
`get_device_info` against 11.8 ms and 13.1 ms, and 25 fresh connections across
a warm phase where both interfaces opened none. The same absence shows up in
execution, where the native path re-reads the dynamic architecture inside
`run_circuit` while the interfaces serve it from cache.

This is worth stating plainly because the framing of an interface layer is
usually what it costs. Here, on the most common access pattern — ask the same
device something more than once — both interfaces are roughly forty times
cheaper than talking to the provider directly, because both introduced a cache
the provider client does not have.

The difference between the two interfaces was entirely in cold start, and it was
large: 1334.6 ms for QRMI against 3376.8 ms for QDMI-on-IQM under identical
conditions, with the native client at 2831.1 ms in the same three-arm run. That
is the cost a component pays when it must observe from a fresh process — a
per-job scheduler hook, a monitoring probe, a short-lived task — which is
precisely the pattern a resource manager uses.

**That gap has since closed.** Re-measured on the same path at iqm-qdmi 1.4.0,
QDMI-on-IQM is 1526.8 ms against a QRMI control at 1452.0 ms, about 5% apart.
The paragraphs below diagnosed the cause as connection handling rather than
interface design and said it was fixable by retaining a session across requests.
That is what upstream did, and the numbers moved as predicted. The reasoning is
left standing rather than rewritten, because the diagnosis holding up is the
useful part, and because the native-client arm was not re-run so the three-way
comparison above is still the only one measured.

Note where the native baseline falls: slower than QRMI, close to QDMI-on-IQM.
Neither interface imposes a cold-start penalty over talking to the provider
directly. QRMI is faster than the native client cold, for the same reason it is
faster than QDMI-on-IQM — it reuses one connection where the others do not.

**The cause is transport handling, not interface design.** The gap is not
explained by request count, which is three against roughly five. It is
explained by connection reuse: one pooled connection and one TLS handshake
against five connections and five handshakes. This is a property of the
QDMI-on-IQM implementation, fixable there by retaining a session across
requests, and it should not be read as a property of the QDMI interface.

That claim has now been tested rather than argued. iqm-qdmi 1.4.0 added a
per-device connection pool, the handshake count went from five to one, and the
cold gap went with it. An implementation detail carried the whole of a
difference that could easily have been attributed to the interface, which is
worth remembering the next time a comparison of this kind produces a clean
number.

**Why this was measured over a wide-area path, and why that is not a
disclaimer.** These numbers were taken from outside the site, over an SSH
tunnel to the device. That inflates every round trip, and on a local network
the absolute figures and the ratio would both be smaller. It would be a mistake
to treat the wide-area case as an artifact to be corrected away: remote use of
quantum resources is an expected deployment, not an anomaly. Instruments are
shared between sites, hybrid workflows reach devices that are not local to the
compute, and provider-hosted devices are reached over the public internet by
construction.

Under that condition the connection-handling difference stops being a
micro-optimization. Four extra TLS handshakes are close to invisible at
sub-millisecond round-trip time and dominant at hundreds of milliseconds. The
same implementation choice is therefore negligible locally and
deployment-limiting remotely, and a comparison run only on a local network
would not have surfaced it at all. Latency-sensitive differences deserve to be
measured under latency, precisely because that is where they decide whether a
usage pattern is viable.

**For a common spec.** Four requirements follow.

First, provider-reported timing must survive the interface. The native arm
shows the phases are already there — queue wait, validation, compilation,
execution — so the requirement is passthrough with a defined shape, not a new
timing model. An interface that reduces a job result to counts plus a status
has thrown away the only data that distinguishes a busy device from a slow
path.

Second, queue and load state as first-class telemetry, so admission and
scheduling decisions can be made on observed device state rather than inferred
from failures.

Third, transport expectations stated in the specification rather than left to
each provider implementation — connection reuse in particular. The observation
cost of an interface is part of its contract with a resource manager, and
leaving it unspecified produced a 2.5x difference between implementations of
the same interface, concentrated exactly where remote deployments are most
sensitive.

Fourth, caching belongs in the contract too. Both interfaces added one and the
provider client did not, which is why both are dramatically cheaper on repeat
observation — but neither states a policy, so a caller cannot know whether a
value is fresh, how stale it may be, or how to force a re-read. Today that is
an accident of implementation that happens to favour the interfaces. A
specification should say what is cached, for how long, and how to bypass it,
because a scheduler acting on calibration data needs to know whether it is
reading the device or a memory of it.

</details>

## Device Authentication

Device authentication is how each interface obtains and applies the credentials
used to reach the quantum hardware provider. This axis focuses on how the
credential is injected and when it is required.

<details>
<summary><strong>QRMI Behavior</strong></summary>

For an IQM resource, QRMI reads credentials from environment variables at
resource construction: `{backend}_QRMI_IQM_ISA_ENDPOINT` for the server URL and
`{backend}_QRMI_IQM_ISA_TOKEN` for the bearer token, with an optional
`{backend}_QRMI_JOB_ACQUISITION_TOKEN`. The names are keyed by the backend
portion of the resource id. In a SLURM deployment the SPANK plugin injects these
into the job environment during the prologue. The token is applied as an HTTP
bearer token on the IQM Server API calls.

Credentials are read once, at construction, from the process environment. The
introspection path (`target()`) is not session-bound, so it needs only a valid
endpoint and token.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI passes connection settings as device-session parameters rather than
environment variables. In FoMaC these are arguments to the session/device
loader: `base_url`, `token` (or `auth_file`), and `custom1..5`. The IQM
implementation holds a token manager and applies the bearer token to its HTTP
calls.

Authentication is bound to the session. The session must be initialized before
any query, and initialization fails without a base URL and token — after which
every subsequent query returns `QDMI_ERROR_BADSTATE`.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

Both interfaces ultimately authenticate with an IQM bearer token over HTTPS. The
difference is injection and binding. QRMI takes credentials from named
environment variables, which aligns with a scheduler-injected model
(SPANK-populated job environment). QDMI takes them as explicit session
parameters, which aligns with an in-process client model. QRMI's introspection
is not session-bound and needs only the token; QDMI's is session-bound, so
credentials plus a successful session initialization are prerequisites to any
call.

In QFw both are fed from the same device-access resolution, so the same URL and
token reach both interfaces — delivered as environment variables for QRMI and as
function parameters for QDMI.

For a common spec: credential acquisition should be separable from the
execution and introspection APIs (both interfaces already do this), with a
defined injection contract that supports both environment-based
(scheduler-injected) and parameter-based (in-process) delivery.

</details>

## Control-Plane Authorization

How protected service-control calls are authorized, separately from a user's
right to execute work. The distinction matters where one credential lets a user
run a circuit and another lets an operator change policy or inject credentials.

<details>
<summary><strong>QRMI Behavior</strong></summary>

There is one credential and one privilege level. A resource authenticates to
the provider with a bearer token read from `{backend}_QRMI_IQM_ISA_TOKEN` or
its per-backend equivalent, and every call on the trait uses it. No call is
distinguished as privileged, because no call changes site policy — as the
previous two axes record, there is no policy to change.

The genuinely privileged operation in a QRMI deployment happens outside QRMI.
The SPANK plugin runs in the SLURM prologue, as root, and injects credentials
into the job environment. Whether a user may reach a device is decided there,
by the resource manager, and QRMI's role begins after that decision with a
token it does not question.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI carries a richer identity vocabulary and the same single privilege level.
Session parameters include `TOKEN`, `USERNAME`, `PASSWORD`, `AUTHFILE`,
`AUTHURL` and `PROJECTID`, so a session can be established several ways and can
name a project for the provider's accounting.

`QDMI_ERROR_PERMISSIONDENIED` exists among the error codes, which implies an
authorization model — but there are no control-plane calls in either header for
it to protect. In practice it reports the provider refusing a device operation,
not an operator being denied a privileged one.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

Neither interface separates user execution rights from privileged operations,
and today neither needs to, because neither has a privileged operation to
protect. The absence is consistent rather than dangerous: there is no policy
API, no configuration call, and no credential-injection entry point, so a
single device-facing credential is sufficient for everything each interface
actually does.

That consistency is worth stating because it is a constraint on how these
interfaces can grow. The moment either gains a control-plane call — set a
limit, change a queue policy, register a device, inject a credential — a second
authorization axis becomes mandatory, and neither has the vocabulary for it.

One argument from the QDMI side is worth importing here, because it constrains
what this axis can ever ask of a device contract. MQT Core's Slurm tutorial
notes that consulting a more trustworthy source than the mutable job
environment would still not make it an access-control boundary, because a
program can call `driver.open_device(device_id)` directly. A device interface
sits below the enforcement point by construction, so whatever it checks can be
bypassed by calling the layer it is. Authorization has to be enforced by the
provider or the operating system, and a specification that places it in the
device contract is specifying something that cannot hold. See Interface Role
And Scope.
QDMI is marginally better placed, having an identity model and a
`PERMISSIONDENIED` code already; QRMI would be starting from a token read out
of an environment variable.

The deployment already has a control plane, and it is the site's: SLURM decides
who may reach which device, and its prologue injects the credential as root.
That is a real separation of privilege — the user never holds the operator's
rights — and it works precisely because it sits outside both interfaces.

**For a common spec.** If a specification stays at the device-and-resource
level and leaves policy to the site, one device-facing credential remains
adequate and this axis stays small. If it grows any operator-facing surface —
which the previous two axes argue against, but which is a live option — then
authorization must be split at the same time and not retrofitted: protected
calls named explicitly, a privilege distinct from execution rights, and a
defined answer when an unprivileged caller attempts one. The failure mode to
avoid is the one visible under Admission, where a call exists, returns success,
and does not do what its name says.

</details>

## Data Normalization

Data normalization is the conversion from provider-native or interface-native
payloads into common records that downstream software can consume without
provider-specific parsing. This axis focuses on what each interface returns and
what a consumer must still do to reach a common record.

<details>
<summary><strong>QRMI Behavior</strong></summary>

QRMI returns provider-native payloads. `target()` is raw IQM JSON, and
`task_result()` is raw IQM measurement JSON. Conversion to any common record is
the consumer's responsibility. QRMI ships a Qiskit adapter that converts these
into Qiskit objects, but that adapter is a separate layer above the core
`QuantumResource` API, not part of the resource interface itself.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI returns interface-native typed values through its property and result
model, surfaced by FoMaC as typed Python objects. These values are already
vendor-neutral, but they are QDMI's shape (sites, operations, typed job
results), not a downstream common record. A consumer that targets its own record
schema still adapts from the QDMI shape.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

Neither interface emits the consumer's final record; both require an adapter.
They differ in the level they start from. QRMI hands back raw provider data
(thick), which needs a provider-specific adapter. QDMI hands back neutral typed
data (thin), which needs a shape adapter.

In QFw this is implemented as "one schema, two adapters": the QRMI raw IQM data
is normalized by `qhw-iqm`, and the QDMI FoMaC data is normalized by
`fomac_normalize`, and both converge on the same `qhw-*-v1` records. The two
adapters can populate different fields of the same record — for calibration, the
QRMI adapter fills the IQM observation-set identity (set ids, counts,
timestamps) while the QDMI adapter fills measured T1/T2 and gate fidelity.

**The thick source is the fragile one, and the recheck demonstrated it.** The
two adapters are not equally exposed to provider change, and the upgrade made
the asymmetry concrete. QRMI 0.22.0 added `static_quantum_architecture` to the
`target()` document. That is an additive change by any normal reading, and it
broke the QRMI adapter. The field had never been populated, the adapter's
handling for it had therefore never executed, and the new value arrived as a
list where the code expected an object. The first introspection call after the
upgrade raised `AttributeError: 'list' object has no attribute 'get'`.

Nothing equivalent could happen on the QDMI side within a single interface
version, because a new capability there is a new property with a declared type,
which an existing caller does not ask for and so never receives. The thin
source's adapter breaks when the *interface* changes. The thick source's adapter
breaks when the *provider payload* changes, which happens more often, is not
governed by the interface's own versioning, and does not announce itself.

**A second instance landed while this section was being written, one layer
lower, and it did not resolve the way the others did.** A QRMI developer
reported on 2026-08-24 that IQM changed the response format of the Get Health
Status API and deployed it to IQM Resonance, which broke `is_accessible()` in
QRMI's IQM implementation.

The mechanism is worth recording because it is not a consumer bug. QRMI's IQM
client is generated from IQM's OpenAPI document, and the generated
`IqmServerQcHealthStatus` declared both `healthy` and `updated_at` as required
fields rather than options. A provider-side change to either one fails
deserialization inside QRMI itself, and the caller sees the reachability check
error out. Nothing in QRMI's own version number moves when that happens. A site
can hold a library version fixed, change nothing, and have a call stop working.

**QRMI 0.24.0 adapted to the new format, and the interesting part is what that
did rather than what it fixed.** The two shapes are incompatible: the old
response is flat, `{"healthy": ..., "updated_at": ...}`, and the new one nests,
`{"operational": ..., "health": {...}}`. Both fields are required in whichever
model the client carries. So each QRMI version speaks exactly one of them.

The ORNL q20 was checked at both versions. It still serves the old flat shape,
verified again on 2026-08-27. Under 0.23.1 `is_accessible()` returned true
there and failed on Resonance. Under 0.24.0 it fails there, with
`error in serde: missing field 'operational'`, and works on Resonance.

**Upgrading the client did not fix the failure. It moved it to the other
deployment.** There is no released QRMI version that answers this call correctly
against both sites at once, and a site operator has no lever that would produce
one, because the choice is not theirs to make. That is a sharper statement of
the problem than the one this section started with. The behavior of an
interface call is a property of the provider deployment rather than of the
interface version, neither the interface nor its version number lets a caller
tell which deployment it is talking to, and when a provider stages a breaking
change across its own estate, every client version is wrong somewhere for the
duration.

Three lessons for a specification, none about schemas. First, an optional field
that a source has never populated is untested code in every consumer, and the
day the source starts sending it is indistinguishable, downstream, from a
breaking change. Optional-and-absent and optional-and-present should be
exercised before they are relied on. Second, if a normalized record is going to
carry a passthrough of provider-native data, the interface should version that
passthrough, or consumers will keep discovering its shape by crashing. Third,
generated clients over provider APIs inherit the provider's release cadence
regardless of their own. A specification that expects independent
implementations needs to say what a required field means when the provider
stops sending it, and an implementation generated from a provider schema should
treat response fields as optional unless the provider commits to them, because
the alternative is that a remote deployment can break a pinned client.

The staged-rollout case adds a fourth, and it is the one with no obvious answer.
A single client version cannot be correct against a provider that is midway
through changing its own API across sites. Tolerating both shapes is the only
behavior that works everywhere, which means a generated client should accept
either the old or the new schema during a transition rather than exchanging one
for the other. QRMI does this deliberately elsewhere and has the machinery for
it: the IBM resource-type rename in 0.23.0 accepts both the legacy and the new
names until a published date. The health-status change got a replacement
instead of a transition, and the difference is not that one is harder. It is
that one was recognized as a migration and the other was treated as a fix.

For a common spec: the normalized record schema is the contract, and the
normalizer is a per-source adapter. A single schema with clearly optional,
provider-specific fields lets a thick source and a thin source converge on one
record without forcing either to invent data it does not have. The optional
fields are where the cost lands, so the specification should say what a
consumer may assume about a field it has never seen populated, which today is
nothing.

</details>

## Program Representation And Placement

This axis covers the program formats each interface accepts, where execution
options live, and whether logical-to-physical placement survives to the
provider in an interpretable form. The observations below come from the same
live-hardware execution runs as the Job Lifecycle axis; the canonical program
was OpenQASM (`x q[0]; measure q[0] -> c[0]`), transcoded once and submitted
through both interfaces.

<details>
<summary><strong>QRMI Behavior</strong></summary>

The accepted representation is typed by resource, and for IQM it is the
provider's: IQM circuit JSON inside the caller-built run request. There is no
format declaration or negotiation — the resource type implies the format.

Placement is embedded in the program: IQM instructions name physical qubits
directly (`"locus": ["QB1"]`). The run request has a `qubit_mapping` field for
logical-name mapping, but it is unused when the circuit already carries
physical names. In the QFw shim, OpenQASM is parsed with Qiskit and serialized
to IQM instructions by `iqm.qiskit_iqm.serialize_instructions`, and the shim
records the logical-to-physical assignment in circuit metadata. The run request
observed on hardware carried `qubit_mapping: null`, physical loci in every
instruction, and `"logical_to_physical": {"0": "QB1"}` as metadata — placement
intent survives, but as embedded physical names plus an informational echo, not
as a field the provider interprets.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI declares the program format as a typed job parameter
(`QDMI_JOB_PARAMETER_PROGRAMFORMAT`) separate from the program bytes, and the
formats are enumerated. QDMI-on-IQM accepts the provider-native single-circuit
`IQMJSON`, QIR base strings, and calibration programs — so a portable candidate
(QIR) is advertised beside the native form, and the interface has a defined
place to reject an unsupported format at submission.

For the `IQMJSON` format, placement is the same embedded physical naming as on
the QRMI path — the transcoded circuit is identical. An optional qubit mapping
can be attached by the device implementation when it assembles the run-request
envelope, but the caller does not express placement through a portable QDMI
field.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

For IQM, both interfaces ultimately submit the provider circuit form with
physical qubit names, and the hardware run confirms placement survives that way
through both — the same `prx` on `QB1` executed identically. Neither interface
carries placement as a first-class, portable field: QRMI has an unused provider
mapping slot plus a metadata echo, QDMI-on-IQM an implementation-internal
option. Placement is a side effect of serialization in both.

The format story differs. QDMI's explicit, enumerated format declaration gives
it a negotiation point QRMI lacks: QRMI's payload is typed by resource, so
format portability is invisible to the interface itself. This mirrors the
Runtime Submission conclusion — QDMI is structurally closer to an
envelope-plus-payload model.

For a common spec: program format should be a declared, negotiable property of
the submission, and placement should be a first-class envelope field with
defined semantics. Embedded physical names work, but a runtime cannot validate,
retarget, or even reliably read placement that exists only inside the
serialized program.

</details>

## Error Model

The error model is how each interface represents failures, provider errors, and
retryability at the call boundary. It determines whether a caller can detect and
diagnose a failure, or whether a failure is silently absorbed into the returned
data.

<details>
<summary><strong>QRMI Behavior</strong></summary>

**This axis has changed more than any other since the first version of this
document, and almost entirely in one direction. All three of its findings have
been fixed upstream.** The behavior it described is recorded below because the
sequence is instructive, but none of it is current.

**What it used to do.** QRMI's methods propagated most failures as exceptions,
with the Python binding converting an internal `anyhow` error into a bare
`PyRuntimeError`. The introspection path was the exception. `target()` assembles
its result from three IQM Server REST calls, and each was individually guarded
so a failure was logged and the field replaced with `null`:

```rust
resp["dynamic_quantum_architecture"] = match get_dynamic_quantum_architecture_v1(...).await {
    Ok(bytes) => parse(bytes),
    Err(e)   => { error!("Failed to get dynamic_quantum_architecture: {:?}", e); json!(null) }
};
```

So `target()` returned `Ok` with null fields on partial or total failure, and
the HTTP status never reached the caller. Diagnostics depended on the `log`
crate, `env_logger` was initialized only in the standalone binaries, and there
was no bridge into Python. A failed fetch therefore surfaced as empty data with
no exception and no log line. (Verified against tag `0.17.2`.)

**Null substitution was removed in 0.22.0.** Counting the substitutions in
`src/iqm/server.rs` dates it exactly: six occurrences of `json!(null)` in every
release from 0.17.2 through 0.21.0, and zero from 0.22.0 on. The three required
fetches now propagate with `?`, and the function's own documentation states the
new contract, that it returns `Err` rather than a document with a field
silently replaced by `null`.

One field is a deliberate exception, and it is the right kind. The
`static_quantum_architecture` added in the same release is optional, because
the artifact's availability depends on the quantum computer's Station Control
version, so a 404 for it specifically is still represented as `null` while any
other failure propagates. That is a documented optional field rather than a
swallowed error, which is the distinction the old behavior failed to make.

**A logging backend now exists, and `RUST_LOG` now matters.** 0.22.0 bridged
Rust `log` records into Python's `logging`. `qrmi.logger` is a real stdlib
logger named `qrmi`, a sink is installed at import, and `set_log_callback`
replaces it. The gate is `RUST_LOG`: with it unset nothing is emitted at all,
and with `RUST_LOG=debug` a single `target()` call yields one
`reqwest::connect` record. So the old claim inverts. There is a backend, and
`RUST_LOG` is exactly what governs it.

**Failures became typed in 0.24.0.** Rust gained a `QrmiError` enum with a
`.kind()` method, C gained `qrmi_get_last_error_kind()` and a return-code enum
that grew from three values to twelve, and Python gained an exception hierarchy
under `qrmi.QrmiError`. Four conditions are classified specifically: resource
not found, task not found, authentication failed, and invalid input. Anything
else falls back to a generic `Other`, including HTTP statuses such as 403 that
vendors use inconsistently enough that guessing was judged worse than not.

**Verified on the current version.** The precise failure this axis documented,
a base URL with a trailing slash producing `//api/v1/...`, was re-run against
the ORNL q20 under 0.24.0. It now raises, and it raises classified:

```
target() RAISED: AuthenticationFailedError
  authentication failed: {"error_code":"unauthorized", ...}
```

Under the documented behavior the same call returned a fully-null target with
no exception and no log.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI returns a typed integer status code from every call
(`QDMI_SUCCESS`, `QDMI_ERROR_INVALIDARGUMENT`, `QDMI_ERROR_BADSTATE`,
`QDMI_ERROR_NOTSUPPORTED`, `QDMI_ERROR_FATAL`, `QDMI_ERROR_TIMEOUT`, ...). The
device implementation maps its internal and provider failures onto these codes.
For example, the IQM implementation returns `QDMI_ERROR_BADSTATE` when a device
property is queried before the session is initialized, and
`QDMI_ERROR_INVALIDARGUMENT` for a malformed request.

The status code is returned at the point of the failing call, and MQT Core's
FoMaC layer surfaces a non-success code to the Python caller as a raised error
rather than as substituted data.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

**The gap this axis described has closed, and the two interfaces now sit much
closer together.** QDMI reports failures as typed codes at the call that failed.
QRMI, as of 0.22.0 and 0.24.0, raises on the introspection path instead of
substituting `null`, classifies its failures, and has a logging backend that is
active when it is embedded as a library. Each of those was a separate finding
here and each is now historical.

**The original observations are worth keeping, because they show what the old
behavior cost.** Two were recorded from QFw testing. A configured base URL with
a trailing slash made QRMI's IQM client build `//api/v1/...`, all three
`target()` fetches failed, and the shim received a fully-null target with no
exception and no log. The second went further. With the IQM endpoint
unreachable, `target()` again returned successfully with all three fields null,
and because the QRMI execution path consumes that same cached document to build
its run request, the connectivity failure surfaced two layers up at circuit
transcoding as "IQM dynamic architecture did not report active qubits". An
availability fault presented as a device-data fault. The null substitution did
not only degrade introspection, it fed misleading state into submission.

That second case is the argument for why silent degradation is worse than it
first looks. A null field is not a smaller version of an error. It is an error
converted into plausible data, and plausible data travels.

**Two things are worth drawing out of the fix rather than just noting it.**

The first is about who found it. This behavior was visible from a consumer, and
it was visible because the consumer was a *shim* spanning two libraries, so the
same failure could be watched through both paths and only one of them lied
about it. A single-library integration would have seen empty data and had
nothing to compare it against. That is the shim earning its keep as a learning
vehicle rather than as a piece of infrastructure.

The second is about sequencing. The three fixes landed in the order
raise-then-classify: 0.22.0 made failures visible, 0.24.0 made them
distinguishable. That is the right order and it is worth stating as a
recommendation, because a specification that mandates a rich taxonomy of error
kinds before it mandates that errors be raised at all has optimized the wrong
half. The taxonomy is only reachable once the failure stops being swallowed.

For a common spec, the requirements are unchanged, and the point is that they
are now met by both implementations rather than one. An introspection or target
call needs a defined error-propagation contract, a raised error or a typed
code. Provider adapters must not substitute empty data for a failed fetch, and
where a field is genuinely optional, that must be declared as optionality
rather than expressed as a swallowed failure indistinguishable from it. And a
logging facility has to be active when the interface is embedded as a library,
not only in its standalone binaries. QRMI's `static_quantum_architecture`,
optional by documentation and null only on a 404, is the shape the middle
requirement is asking for.

</details>

## Extensibility And Versioning

How does each interface grow, how does it advertise an optional feature, and
what breaks when it changes? The two answers differ sharply, and — as with
several axes here — the difference follows from where each sits: QDMI must keep
an ABI stable because vendors ship libraries independently, while QRMI carries
its vendors in-tree and so has never needed to.

<details>
<summary><strong>QRMI Behavior</strong></summary>

Growth happens by editing the interface. The two extension points are closed
Rust enums — `ResourceType` with seven backends and `Payload` with four
variants — so supporting a new machine means adding a variant, and adding a
variant means releasing QRMI. Downstream code that matches exhaustively on
either sees a breaking change.

The `QuantumResource` trait is a single surface of twelve methods, and every
resource implements all of them. There is no way to implement a subset, and no
declaration that a method is unsupported for a given backend, which is the
mechanism behind the `acquire()` behaviour recorded under Admission: a resource
with no session concept still has to return something, so it returns a
generated identifier.

There is no runtime version discovery. A caller cannot ask a resource which
QRMI version it implements, and there is no capability query to probe an
optional feature with. Combined with the error model — failures surface as
exceptions carrying strings rather than typed codes — a caller cannot reliably
distinguish "this backend does not support that" from "that call failed".

The one place QRMI does version explicitly is the payload. The repository ships
`qrmi_payload_v1_schema.json`, a JSON Schema with four `oneOf` variants
distinguished by their required fields (`program_id` + `parameters`, `job_runs`
+ `sequence`, `human_qir` + `input_params`, `iqmjson` + `job_type`). Note that
the file is named `v1` while the schema's own `version` field reads `0.4.0`. It
versions the payload, not the interface.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

Growth happens by adding property values, not by changing signatures. Every
query goes through a generic accessor — `QDMI_device_query_device_property`,
`_query_site_property`, `_query_operation_property`, `QDMI_job_query_property`,
`QDMI_session_query_session_property` — taking a property enum plus a sized
output buffer. A new property is a new enum value; the function signature, and
therefore the ABI, does not move.

Optional features are answered rather than guessed at. A property an
implementation does not provide returns `QDMI_ERROR_NOTSUPPORTED`, one of
eleven typed codes that also include `NOTIMPLEMENTED`, `NOTFOUND`,
`PERMISSIONDENIED` and `LIBNOTFOUND`. A caller can probe and receive a defined
answer, which is what makes a partial implementation expressible rather than
something to be worked around. **In the version-skew case this degrades, and
that is measured below.**

Vendor-specific extension has reserved space: five `CUSTOM` slots in each
property family — device, site, operation, job and session properties, and job
parameters. QDMI-on-IQM uses them, passing the quantum-computer alias as a
session parameter and exposing the active calibration set ID through
`QDMI_DEVICE_PROPERTY_CUSTOM1`. (An earlier revision of this document described
that slot as carrying raw IQM device data. It does not, and did not at 1.2.0
either. The implementation comment above the line states plainly that the slot
exposes the calibration set ID. The error came from reading a local branch that
had added raw passthrough, and attributing it to the released library.)

Version is discoverable at runtime through the device itself:
`QDMI_DEVICE_PROPERTY_VERSION` for the device and
`QDMI_DEVICE_PROPERTY_LIBRARYVERSION` for the QDMI version its library
implements. A caller that has just loaded an unfamiliar device library can ask
what it speaks before relying on it.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

**One can grow without breaking; the other cannot.** QDMI adds capability by
adding enum values behind unchanging function signatures, so a device library
built against an older header keeps working and an updated caller learns what
is missing through `NOTSUPPORTED`. QRMI adds capability by adding enum variants
and trait methods, both of which are breaking changes, and offers no way to
express partial support.

That needs one qualification after the 0.24.0 recheck. QRMI grew an entire error
taxonomy in that release and got it into Python additively, by having every new
exception subclass the `RuntimeError` its failures were always raised as, so
existing `except RuntimeError` code kept working untouched. The Rust side did
break, since the public signature moved from `anyhow::Error` to a `QrmiError`
enum. So the claim holds where QRMI is consumed as a Rust crate and does not
hold where it is consumed through its Python bindings. The difference is not
luck. It is that the Python surface had a pre-existing supertype to hang the new
types under, which is the same trick QDMI's reserved enum space plays and an
argument for designing that room in from the start. That is the difference between an interface designed
to be implemented by parties who release on their own schedule and one designed
to be edited in place.

**The monolithic trait is the root of several findings elsewhere.** The axis
description asks whether provider implementations can evolve without treating
the interface as one monolith. QRMI's twelve-method trait is exactly that
monolith: every backend must present every method, so methods that are
meaningless for a backend become no-ops rather than honest absences. The
`acquire()` divergence under Admission and the missing capability advertisement
under Device Discovery are not three separate oversights — they are one design
choice observed from three directions.

**Reserved custom slots are extensibility without portability, and there is now
a concrete case.** QDMI's five `CUSTOM` slots per family let a vendor expose
anything without a spec change, which is genuinely useful. But a custom slot
means whatever its vendor decided, so portable code cannot use one, and two
vendors will not agree on `CUSTOM1`. They are an escape hatch, and the more load
they carry the less the neutral model is actually describing the device.

This axis previously offered "watch what accumulates in them" as a heuristic for
finding what the specification is missing. Applying it produces a specific
answer. What accumulated in QDMI-on-IQM's `CUSTOM1` is the **calibration set
ID**. That is not a vendor peculiarity. Every provider that calibrates has one,
and it is the field that makes any measured value interpretable after the fact,
because a fidelity without the calibration it came from cannot be compared to
anything. It sits in a numbered slot because the specification has no named
place for it. The heuristic worked, and the backlog it produced has one clear
item on it.

The reverse case is visible in the same window and confirms the mechanism.
Queue position and queue length were also missing from the named model. QDMI
1.3.3 added `QDMI_JOB_PROPERTY_QUEUEPOSITION` and
`QDMI_DEVICE_PROPERTY_QUEUELENGTH` as named properties, without moving any
function signature, and MQT Core bound them one minor release later. That is the
promotion path working as designed: identify what vendors are carrying out of
band, name it, and the ABI does not move.

**The typed not-supported answer does not survive version skew, and this was
measured.** The claim above is that a caller can probe an unknown implementation
and receive a defined answer. Upgrading the client side to MQT Core 3.9, built
against QDMI 1.3.3, while the device library remained QDMI-on-IQM 1.3.0, built
against QDMI 1.3.2, produced a case where it does not.

**This particular instance has since closed. iqm-qdmi 1.4.0 moved the device
library to QDMI 1.3.3, so both sides now agree and the call below succeeds.**
The finding is kept rather than deleted, for a reason that matters more than the
individual bug. What fixed it was the device library catching up, which is
exactly the thing an ABI model exists to avoid having to require. Nothing about
the error-reporting contract changed. Re-open any two components at different
QDMI versions and the same confusion returns, and the whole point of shipping
device libraries independently is that they will be at different versions.

Asking the q20 for `Device.queue_length()` returns
`QDMI_ERROR_INVALIDARGUMENT`, surfaced in Python as
`ValueError: Querying QUEUE LENGTH: Invalid argument`. Not
`QDMI_ERROR_NOTSUPPORTED`. The mechanism is straightforward once seen.
`QDMI_DEVICE_PROPERTY_QUEUELENGTH` is new in 1.3.3, so it sorts above the
`QDMI_DEVICE_PROPERTY_MAX` the device library was compiled against. That
library's property handler rejects anything at or above its own `_MAX`, other
than the `CUSTOM` slots it names explicitly, as an invalid argument before any
per-property logic runs. From the caller's side two different situations now
return the same code: passing a genuinely malformed property, and asking a
slightly older library about a property it has never heard of.

This is the exact case the ABI model exists to handle. A device library released
by a vendor on its own schedule will routinely be older than the client asking
it questions. One patch release of drift was enough to produce this. The problem
is not that the older library lacks the feature, which is expected and fine. It
is that `NOTSUPPORTED` means "I know this property and do not provide it" while
the unknown-enum case falls through to a generic argument error, so a caller
cannot distinguish "this device has no queue" from "this device predates the
question" from "you passed garbage". A conformance requirement that unknown
values in the reserved property range return `NOTSUPPORTED` rather than
`INVALIDARGUMENT` would cost an implementation one comparison, and would make
capability probing work across versions, which is the thing it is for.

The upgrade that closed this is itself the argument for the requirement. A
caller could only recover by waiting for the vendor to ship a library built
against the newer specification. That is precisely the coupling the ABI and the
typed error were meant to remove, and for the duration of the gap they did not
remove it.

**Runtime version discovery is a real asymmetry.** QDMI lets a caller ask a
just-loaded library which version it implements. QRMI has no equivalent,
because the question does not arise when the vendor code ships in the same
build — which is consistent, but means anything reusing QRMI's model with
independently-released backends would have to add it. Version discovery is also
the workaround for the finding above. A caller that cannot trust the error code
has to ask each library its version and keep its own table of which properties
exist in which release, which is precisely the out-of-band knowledge the typed
error was supposed to make unnecessary.

**For a common spec.** If the provider side is an ABI implemented by others —
which the Interface Role axis argues is the right shape — then QDMI's
mechanisms are close to the minimum needed: a generic property query so new
capability does not move the ABI, a typed not-supported so partial
implementations are expressible and discoverable, runtime version reporting,
and reserved vendor space. To that should be added what neither has: a stated
policy for what happens when a caller is newer than an implementation, and the
discipline of treating accumulation in the custom slots as a backlog for the
next revision rather than as a permanent home. Both of those moved from
hypothetical to observed in this recheck. The newer-caller policy has a measured
failure, one patch release of drift returning the wrong error code. The
custom-slot backlog has a named first item, the calibration set ID. And whichever model is chosen,
capability must be declarable — the recurring cost across this document is that
a caller cannot ask what a resource actually supports before depending on it.

</details>
