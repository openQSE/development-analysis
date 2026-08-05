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
| `qrmi` | 0.17.2 | QRMI interface and its IQM resource implementation |
| `iqm-qdmi` | 1.2.0 | QDMI-on-IQM, the QDMI device implementation for IQM |
| `mqt-core` | 3.7.0 | QDMI headers and the FoMaC layer the Python caller uses |
| `iqm-client` | 34.0.1 | IQM Server client used by the QFw drivers |
| `iqm-station-control-client` | 12.1.1 | IQM station-control models |
| `iqm-pulse` | 13.0.1 | IQM circuit objects produced by transcoding |
| `qhw-iqm` | 0.1.0 | normalization of IQM data into `qhw-*` records |

Hardware observations are against the ORNL IQM 20-qubit device, reached through
the QFw shim (`openQSE/QFw`, `services/svc_lib_qpm`).

On source citations. Where this document cites QDMI-on-IQM source it means the
corresponding upstream tag, not any local working tree — `iqm-qdmi` ships as a
wheel containing a compiled device library, so a checkout used for reading is
not necessarily the code that ran.

Citations are anchored by symbol or function name rather than by line number
wherever the file is one that moves. Line numbers were tried first and did not
survive: every QDMI-on-IQM line reference in Calibration And Quality Data had
drifted by the time it was rechecked at 1.2.0, and the QFw ones had been moved
by later commits to those same files. The `qrmi` references retain line numbers
because they were verified against the pinned 0.17.2 tag, which is not moving.

**Status (2026-07-28):** the comparison now has live-hardware backing. Both
interfaces ran against the ORNL IQM 20-qubit system through the QFw shim:
device introspection through each returned the same normalized topology (20
qubits, 30 edges, in agreement), and one canonical circuit executed through
each returned identical counts in the same normalized result record. Axis
entries below that cite hardware observations derive from those runs.

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

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

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
`IBMDirectAccess`, `IBMQiskitRuntimeService`, `IBMQuantumSystem`,
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

Discovery is a query against the session. `QDMI_SESSION_PROPERTY_DEVICES`
enumerates the devices a session can see, surfaced in MQT Core's FoMaC as
`Session.get_devices()`. Devices reach a session through the driver's device
libraries; in the QFw shim the IQM library is loaded explicitly by path with
`add_dynamic_device_library`, which returns the device directly.

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

**For a common spec.** Discovery should be a call, with the resource manager
able to scope its results, so the same API serves both the constrained batch
case and the exploratory client case. Capability advertisement should be
queryable per resource rather than implied by a type constant, should cover
accepted program formats explicitly, and should use a typed
not-supported answer so that unsupported and unavailable are distinguishable.
The test is whether a portable caller can select a device and construct a valid
request without holding out-of-band knowledge about the backend — which today
it cannot do through either interface alone.

</details>

## Admission

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

</details>

## Admission Control Configuration

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

</details>

## Device Scheduler Control

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

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

Calibration identity is not. No job or device property returns the calibration
set a job ran under — the same accessor gap already noted for calibration
identity under Device Introspection. The run-request envelope was assembled
inside the device implementation (see Runtime Submission), so the caller never
held the submission document either, and no provenance of what was actually
submitted is retrievable from the job.

Timing is also absent. The FoMaC job object exposes no submission, queue, or
execution timestamps, so any duration on this path is measured by the caller
around its own calls rather than reported by the provider.

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

On a live IQM server (the ORNL q20), the assembled document contained exactly
those three components and no static architecture: the qubit set `target()`
reports is the dynamic — currently calibrated — one. A consumer that needs the
chip's static topology independent of the active calibration set (a separate
endpoint in the native IQM API) does not get it through this call.

`target()` is not reservation-bound; it does not require `acquire()` and reads
the current data with only a valid endpoint and token. There is no separate
typed topology or property API distinct from this document; discovery and
introspection are the parsing of the raw payload.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

QDMI exposes introspection as a typed, vendor-neutral query interface. Through
MQT Core's FoMaC (`mqt.core.fomac`), a `Device` provides typed accessors:
`name()`, `version()`, `status()`, `qubits_num()`, `supported_program_formats()`,
`needs_calibration()`, `duration_unit()`, `coupling_map()`, `sites()`, and
`operations()`. A `Site` exposes `name()` (the device's real qubit label, e.g.
`"QB1"`), `index()`, `t1()`, `t2()`, and coordinates. An `Operation` exposes
`name()`, its loci through `sites()` / `site_pairs()`, `fidelity()`,
`duration()`, and `idling_fidelity()`.

The values are the device's real labels and metrics, not a provider-specific
document. Every query is session-based: the session must be initialized before
any property can be read.

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

QRMI presents introspection as one raw provider document behind a single call;
QDMI presents it as a set of typed, vendor-neutral properties. The QRMI form
preserves everything the provider sends — including calibration and quality
identity — at the cost of provider-specific parsing. The QDMI form is portable
and directly consumable, but exposes what its neutral property set covers: the
per-qubit and per-gate metric values are present, while the provider's
calibration-set identity is not currently reachable through the FoMaC Python
accessors.

A concrete consequence in QFw: `get_device_info` and `get_coupling_graph` are
served by both interfaces, but `get_backend_info` and `get_dynamic_backend_info`
are served only by QRMI, because their shape carries raw IQM architecture data
(static architecture, active qubits, calibration-set id) that the QDMI-neutral
property model does not expose in that form.

For a common spec: a typed, neutral property model (QDMI-like) is the more
portable basis, provided it also reserves a defined place for provider identity
and provenance (such as the calibration-set id) that the raw model (QRMI)
already carries.

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

- QFw opens the IQM QDMI shared library through MQT Core's FoMaC loader
  (`QFw/services/svc_lib_qpm/drivers/qdmi_driver.py`, `_device()`).
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
returns measurement JSON and no timestamps, so no provider-side timing reaches
the caller (see Job Lifecycle And Results).

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

Cost of observing. The IQM device library fetches during session init, so the
cost is paid when the device is opened; property queries afterwards are local
reads. Measured on the same device and path: 3376.8 ms median cold (5 samples,
3193-3397 ms), then 12-17 ms for repeat queries, no network. Session init opens
**five separate TCP connections** — five TLS handshakes. QDMI-on-IQM issues each
request through cpr's free-function API (`cpr::Get` / `cpr::Post` in
`src/internal/http_client.cpp`), which constructs and destroys a session, and
with it the underlying libcurl handle and its connection cache, per call. No
session is retained across requests, so each one reconnects.

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
depth, queue position, device load, or any provider timestamp. A scheduler
cannot ask either interface how busy the device is, or learn afterwards how
much of a job's elapsed time was queueing rather than execution.

**That gap is not the provider's.** The native client reads the IQM job
timeline and reports the phases directly: queue wait, validation, compilation,
execution, post-processing, and a server-side total. A measured run gave 34.9
ms of queue wait and 107.3 ms of execution on a circuit whose client-side
elapsed time was several seconds. The same device, through either interface,
reports none of it.

So queue-versus-execution is not information that has to be invented for a
common spec, nor obtained by instrumenting callers. It exists at the provider
and is discarded in the layer above. QRMI's `task_result` carries measurement
JSON and drops the timeline that accompanied it; QDMI's job object exposes no
timestamp property at all. That is a stronger and more actionable finding than
a symmetric absence: the requirement is to preserve what the provider already
sends, not to construct something new.

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

The difference between the two interfaces is entirely in cold start, and it is
large: 1334.6 ms for QRMI against 3376.8 ms for QDMI-on-IQM under identical
conditions, with the native client at 2831.1 ms in the same three-arm run. That
is the cost a component pays when it must observe from a fresh process — a
per-job scheduler hook, a monitoring probe, a short-lived task — which is
precisely the pattern a resource manager uses.

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

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

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

For a common spec: the normalized record schema is the contract, and the
normalizer is a per-source adapter. A single schema with clearly optional,
provider-specific fields lets a thick source and a thin source converge on one
record without forcing either to invent data it does not have.

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

QRMI's `QuantumResource` methods propagate most failures as exceptions: the
Python binding converts an internal `anyhow` error into a `PyRuntimeError`. The
device-introspection path is an exception to this rule. For an IQM resource,
`target()` assembles its result from three IQM Server REST calls
(`dynamic-quantum-architecture`, the calibration set, and quality metrics), and
each call is individually guarded so that a failure is logged and the field is
replaced with `null`:

```rust
resp["dynamic_quantum_architecture"] = match get_dynamic_quantum_architecture_v1(...).await {
    Ok(bytes) => parse(bytes),
    Err(e)   => { error!("Failed to get dynamic_quantum_architecture: {:?}", e); json!(null) }
};
```

`target()` therefore returns `Ok` with null fields on partial or total failure;
the HTTP status is never propagated to the caller.

Diagnostics depend on the `log` crate (`log::error!`). In QRMI, `env_logger` is
initialized only in the standalone example and CLI binaries, not in the Python
extension, and there is no `pyo3-log` bridge. When QRMI is used as a Python
library, no logging backend is registered, so those `error!` records are
dropped and `RUST_LOG` has no effect. A failed device fetch then surfaces as
empty data with no exception and no log line.

(Verified against QRMI tag `0.17.2`, the version pinned in the QFw container.)

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

The two interfaces sit at opposite ends of error visibility. QDMI reports
failures as typed codes at the call that failed. QRMI reports execution failures
as exceptions but, on the introspection path, degrades to `null` data instead of
raising; combined with the absence of a logger in its Python build, the
underlying cause is invisible to the caller.

This was observed directly in QFw testing. A configured base URL with a trailing
slash caused QRMI's IQM client to build `//api/v1/...`; all three `target()`
fetches returned 404; and the shim received a fully-null target with no
exception and no log. The same class of failure on the QDMI path would surface
as a non-success status at the query.

A second instance, from the live-hardware runs, extends the consequence into
execution. With the IQM endpoint unreachable (a dropped SSH tunnel in the
remote-access setup), all three fetches failed at the TCP level and `target()`
again returned successfully with all three fields null. The QRMI execution path
consumes the same cached `target()` document to build its run request, so the
connectivity failure surfaced two layers up, at circuit transcoding, as "IQM
dynamic architecture did not report active qubits" — an availability fault
presenting as a device-data fault. The null-substitution therefore does not
only degrade introspection data; it feeds misleading state into submission.

For a common spec: an introspection/target call needs a defined
error-propagation contract (a typed error or a raised exception), and provider
adapters should not silently substitute empty data for a failed fetch. A related
requirement is observability — a logging facility that is active when the
interface is embedded as a library, not only in its standalone binaries.

</details>

## Extensibility And Versioning

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

</details>
