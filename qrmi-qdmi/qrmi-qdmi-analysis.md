# QRMI And QDMI Comparison

This document captures the QRMI and QDMI comparison being performed through
the QFw shim QPM service. The goal is to compare the two interfaces as active
implementations behind one QPM-facing contract. The comparison is anchored in
the current QFw work, where `svc_lib_qpm` can route selected QPM calls to QRMI
or QDMI and return the observed behavior through the same client-side test.

The comparison follows the API-category direction discussed in
`openQSE/openqse-spec` discussion 31. A quantum resource interface should not
be treated as one monolithic API. It has several consumers, including
applications, resource managers, schedulers, operators, monitoring services,
and authentication services. Each consumer needs a different part of the
interface. The comparison below uses those functional blocks as the primary
axes.

## Summary Table

| Comparison axis | QRMI behavior in current QFw shim | QDMI behavior in current QFw shim |
|---|---|---|
| Interface role and scope | Acts as the execution and reservation-oriented path. QFw treats it as the execution owner for the shim resource. | Acts as the device-introspection path. QFw routes selected device and calibration queries through QDMI/FoMaC. |
| Device discovery and capability advertisement | Exposes enough target data for device introspection through `QuantumResource.target()`. QFw currently uses a static descriptor to advertise QRMI coverage. | Exposes device structure through QDMI/FoMaC query objects. QFw currently uses the same static descriptor to advertise QDMI coverage. |
| Admission | Not yet exposed as a QFw-tested API category in the shim path. QRMI is closer to this layer because it owns resource-management concepts. | Not yet exposed as a QFw-tested API category in the shim path. QDMI is currently used for device queries rather than admission decisions. |
| Admission control configuration | Not currently implemented in the QFw shim path. | Not currently implemented in the QFw shim path. |
| Device scheduler control | Not currently implemented in the QFw shim path. | Not currently implemented in the QFw shim path. |
| Runtime submission | Wired as the execution owner. `async_run` routes circuit execution to QRMI in the descriptor. | Not wired for runtime submission in the current descriptor. |
| Job lifecycle and results | Intended owner for status, result retrieval, timing, and metadata after submission. The current shim test exercises `async_run` and `get_last_job_metadata`. | Not currently the job lifecycle owner in QFw. |
| Device introspection | Uses QRMI target data, then normalizes IQM-native data through `qhw-iqm`. | Uses QDMI/FoMaC device objects, then normalizes extracted topology through `qhw-data` builders. |
| Calibration and quality data | QRMI target data appears to carry IQM dynamic architecture, calibration set, and quality metrics. QFw currently wires only part of this path. | QDMI-on-IQM fetches IQM calibration quality metrics internally and exposes selected data through FoMaC properties. QFw currently has only partial binding for calibration snapshots. |
| Telemetry | Not yet separated as a QFw shim API category. Some execution timing can be inferred from QFw result metadata and QRMI job metadata once implemented. | Not yet separated as a QFw shim API category. Device health and status are available through QDMI properties, but QFw does not yet expose a telemetry API. |
| Device authentication | Uses endpoint and token environment variables or shared QFw device-access config to initialize the QRMI IQM resource path. | Uses the same QFw endpoint and token sources to initialize the QDMI-on-IQM FoMaC device session. |
| Control-plane authorization | Not implemented. No protected operator-facing control API exists in the current QFw shim. | Not implemented. No protected operator-facing control API exists in the current QFw shim. |
| Data normalization | Normalizes IQM-native target data through `qhw-iqm`, then returns `qhw-data` records. | Normalizes the FoMaC-extracted representation directly with `qhw-data`. The lower QDMI-on-IQM layer receives IQM JSON internally, but QFw does not currently consume that raw JSON. |
| Program representation and placement | Current QFw smoke path sends OpenQASM through `async_run`. QRMI execution binding remains the intended route for circuit execution and placement behavior. | QDMI-on-IQM supports IQM JSON and QIR-style job parameters internally, including qubit mapping parameters. QFw does not currently route smoke execution through QDMI. |
| Error model | Errors propagate through QFw/DEFw exceptions and shim result dictionaries. Library-specific error normalization is not yet defined. | Errors also propagate through QFw/DEFw exceptions. QDMI-specific status codes are hidden behind FoMaC/Python exceptions in the current QFw path. |
| Extensibility and versioning | Coverage is expressed in the QFw shim descriptor. API-level versioning is not yet split into independent specs. | Coverage is expressed in the same QFw shim descriptor. QDMI can implement more categories without requiring QFw to treat the interface as monolithic. |

<details>
<summary><strong>Interface Role And Scope</strong></summary>

The current QFw shim treats QRMI and QDMI as two lower-level implementations
behind one QPM-facing service. The QPM client does not import either library
directly. It discovers the shim QPM through the QFw resource manager, connects
to that service, and calls the `api_qpm` methods.

QRMI is currently the execution owner in the shim descriptor. This matches the
direction of QRMI as a resource-management interface. The QFw path uses QRMI
for the execution-family calls because those calls need one owner for
submission, status, metadata, and results.

QDMI is currently used as a device-introspection path. QDMI-on-IQM opens a
device session, queries the IQM backend, and exposes the device through the
FoMaC object model. QFw then reads that model and builds normalized device and
coupling records.

The unification path is a QPM-facing contract split into functional API
categories. QRMI and QDMI can implement different subsets. QFw should route by
capability and by resource descriptor instead of assuming that one library owns
the whole quantum resource interface.

</details>

<details>
<summary><strong>Device Discovery And Capability Advertisement</strong></summary>

The current QFw comparison relies on a static per-resource descriptor in
`services/svc_lib_qpm/descriptor.py`. For `ornl-iqm-20q`, the descriptor wires
both `qrmi` and `qdmi`, sets QDMI as the introspection preference, and sets
QRMI as the execution owner.

QRMI coverage currently includes device information, coupling graph,
backend information, circuit execution, job timing, and job metadata. The
implemented QRMI driver path is strongest for `get_device_info` and
`get_coupling_graph`, where it reads `QuantumResource.target()` and passes the
IQM-native payload into `qhw-iqm`.

QDMI coverage currently includes device information, coupling graph, backend
information, dynamic backend information, and calibration snapshot. The
implemented QDMI driver path is strongest for `get_device_info` and
`get_coupling_graph`, where it reads the FoMaC device object.

The unification path is dynamic capability advertisement. A device service
should advertise which API categories it implements, which data formats it can
return, and which calls are complete enough for production use. Static
descriptors are useful during the QFw comparison phase, but the final interface
should let the service report this coverage directly.

</details>

<details>
<summary><strong>Admission</strong></summary>

Admission is the resource-manager-facing decision path. A site scheduler or
resource manager asks whether a proposed job can receive access to a quantum
device. The request needs enough information for the device-side policy to
estimate capacity. Useful inputs include expected circuit count, qubit count,
depth, shot count, one-qubit gate count, two-qubit gate count, and expected
runtime.

The current QFw shim does not expose admission as a QRMI or QDMI test axis.
QRMI is conceptually closer to this layer because it already targets resource
management. QDMI is currently focused on device representation and submission
mechanics.

The unification path is a separate `api_admission` category. It should return
a structured decision that a SLURM, Flux, QRMI, SPANK, GRES, or HRES
integration can translate into its own reservation lifecycle. The decision
should distinguish accepted, rejected, delayed, and accepted-with-limits cases.

</details>

<details>
<summary><strong>Admission Control Configuration</strong></summary>

Admission control configuration is an operator-facing API category. It selects
and tunes the device admission policy. Examples include unlimited admission,
rate-limited admission, time-credit admission, account limits, reservation
limits, and site-specific policy modules.

Neither QRMI nor QDMI exposes this through the current QFw shim. The QFw
prototype also does not yet define the protected control API required to
configure admission policy safely.

The unification path is a separate `api_admission_control` category. It should
be callable by trusted site automation, not by ordinary application code.
Authentication and authorization for this API should be handled separately from
device authentication.

</details>

<details>
<summary><strong>Device Scheduler Control</strong></summary>

Device scheduler control configures the local QPU scheduling policy. It is
used by site operators or trusted automation to select policies such as FIFO,
priority, round robin, shortest job first, longest job first, shot slicing, or
deadline-aware scheduling.

The current QFw shim does not expose scheduler control through QRMI or QDMI.
The existing QPM execution path accepts work through `sync_run` or `async_run`.
Any device scheduling policy should live behind those calls rather than
requiring applications to pick the next task.

The unification path is a protected scheduler-control API. It should configure
the local scheduler and expose queue inspection. Execution calls remain simple
for applications. They submit work. The service schedules the work according
to the configured policy.

</details>

<details>
<summary><strong>Runtime Submission</strong></summary>

Runtime submission is the application-facing execution path. In QFw, this is
represented by QPM calls such as `async_run(info)` and `sync_run(info)`. The
current shim smoke test uses `async_run` with a small OpenQASM circuit.

QRMI is the current execution owner. The descriptor routes `run_circuit`,
`get_last_job_timing`, and `get_last_job_metadata` to QRMI. This makes QRMI the
path that should own job submission and subsequent execution-family state.

QDMI execution is not enabled in the current QFw descriptor. QDMI-on-IQM does
contain job submission logic internally. It can construct an IQM job payload,
include calibration set ID, shot count, execution options, and qubit mapping,
and submit that payload to IQM. The QFw shim has not yet exposed that as the
runtime owner.

The unification path is a common runtime API category. It should define the
submission envelope, accepted program formats, execution options, placement
hints, async completion behavior, and result retrieval contract. QRMI and QDMI
can then be compared by running the same QFw runtime test through each path
when both support execution.

</details>

<details>
<summary><strong>Job Lifecycle And Results</strong></summary>

Job lifecycle includes status, cancellation, completion, result retrieval,
execution timing, provider metadata, and errors. In the QFw shim, this is tied
to the execution owner because stateful job calls must follow the same lower
library that submitted the job.

QRMI is the current owner for this area. The smoke test registers a completion
callback, calls `async_run`, waits for a result event, and can then call
`get_last_job_metadata`.

QDMI is not currently the job lifecycle owner in QFw. Its C++ implementation
can submit jobs and query IQM job results internally, but the QFw shim does
not route job lifecycle calls to QDMI.

The unification path is to define lifecycle results independently from the
submission mechanism. The normalized result should separate user-visible
counts, timing, provider metadata, raw provider payload when requested, and
error details.

</details>

<details>
<summary><strong>Device Introspection</strong></summary>

Device introspection covers device identity, qubits, connectivity, operations,
supported loci, backend identity, and supported program formats.

QRMI currently reads IQM-native target data through `QuantumResource.target()`.
The QFw QRMI driver converts that payload into the input shape expected by
`qhw-iqm`, then returns normalized qhw device and coupling records.

QDMI currently reads device information through MQT Core FoMaC. QDMI-on-IQM
fetches IQM REST data internally, parses it, and populates FoMaC/QDMI device
objects. QFw reads those objects and builds qhw device and coupling records
with `qhw-data` builders.

The unification path is for both implementations to satisfy the same
device-introspection data schema. They do not need to expose the same internal
source representation. The contract should define the normalized output and
the required provenance fields.

</details>

<details>
<summary><strong>Calibration And Quality Data</strong></summary>

Calibration and quality data includes calibration set IDs, quality metric set
IDs, T1 and T2 values, gate fidelity, readout fidelity, calibration validity,
and measurement context.

QRMI target data appears to carry IQM dynamic architecture, calibration set,
and quality metrics. The current QFw QRMI driver only wires the device and
coupling normalization path. Calibration snapshot support is not yet fully
bound in the shim.

QDMI-on-IQM fetches IQM calibration quality metrics internally. It stores
selected metrics on FoMaC/QDMI site and operation objects, including T1, T2,
single-qubit fidelity, and two-qubit fidelity. QFw can access some of that
through FoMaC extraction, but the current `get_calibration_snapshot` route is
still marked as a later milestone.

The unification path is a separate normalized calibration schema. Both
interfaces should return calibration summaries and detailed observations in a
common structure. Provider-specific details can live in extensions.

</details>

<details>
<summary><strong>Telemetry</strong></summary>

Telemetry covers health, queue state, load, availability, timing, current
calibration, status transitions, and operational counters. It serves both
runtime policy and observability.

The current QFw shim does not expose telemetry as a separate category. QRMI
can eventually provide resource and job telemetry. QDMI exposes device status
and static device properties, but QFw does not yet translate that into a
telemetry API.

The unification path is an `api_telemetry` category. It should provide compact
state for schedulers and richer state for monitoring services. Data returned
through this API should use normalized records where possible.

</details>

<details>
<summary><strong>Device Authentication</strong></summary>

Device authentication is the mechanism used to obtain access to the hardware
provider. In the current QFw shim, both QRMI and QDMI use the same practical
credential sources. They read `QFW_QC_URL` and `QFW_API_KEY`, or fall back to
the shared QFw device-access configuration.

QRMI converts those values into the endpoint and token environment expected by
its IQM resource path. QDMI passes the URL and token to the FoMaC dynamic
device loader, which initializes the QDMI-on-IQM session.

The unification path is a separate device-authentication API category. A
trusted resource-manager integration should inject job-scoped credentials into
the QPM service. The service should store them in memory by job or lease ID
and use them for device calls. This avoids exposing provider tokens as a
normal application concern.

</details>

<details>
<summary><strong>Control-Plane Authorization</strong></summary>

Control-plane authorization governs who can change admission, scheduler, and
authentication policy. It is different from device authentication. Device
authentication proves access to the hardware provider. Control-plane
authorization proves that the caller is allowed to change service policy.

The current QFw shim does not implement control-plane authorization. The
development path still assumes direct test access to QPM calls.

The unification path is a protected control API. It should require a trusted
identity from the site resource manager, a control daemon, or a comparable
authorization system. Ordinary application clients should not be able to
configure admission policy, scheduler policy, or device credentials.

</details>

<details>
<summary><strong>Data Normalization</strong></summary>

Data normalization is a cross-cutting category. It applies to device records,
coupling graphs, calibration data, telemetry, and execution results.

QRMI currently gives QFw an IQM-native target payload. The QFw QRMI driver
passes that payload to `qhw-iqm`, which produces qhw-data normalized records.

QDMI currently gives QFw a FoMaC object model. QDMI-on-IQM has already parsed
the IQM REST data internally. QFw extracts topology and operation information
from FoMaC and builds qhw-data records directly. That means QDMI is already
closer to returning an implementation-neutral view, but the current QFw path
does not yet use a shared QDMI data-normalization implementation.

The unification path is `api_data_normalization`. The spec should define the
normalized records, the builder API, and the extractor API. Implementations can
use `qhw-data`, `qhw-iqm`, or their own conforming implementation. QFw should
consume the normalized records and preserve source/provenance fields.

</details>

<details>
<summary><strong>Program Representation And Placement</strong></summary>

Program representation covers the accepted input form for quantum work. It
also covers execution options and logical-to-physical placement.

QRMI is currently exercised through QFw with OpenQASM in the smoke test. The
longer-term execution path needs to define how Qiskit, QIR, IQM JSON, and
placement hints are represented at the QPM contract boundary.

QDMI-on-IQM internally supports IQM JSON and QIR-style program formats. It can
include IQM job options such as calibration set ID, shot count, heralding,
move validation mode, dynamical decoupling mode, and qubit mapping.

The unification path is a submission envelope that carries payload format,
payload reference or inline payload, execution options, and placement hints.
The lower implementation can translate that envelope into its native job
format.

</details>

<details>
<summary><strong>Error Model</strong></summary>

Errors currently flow through QFw/DEFw exceptions and shim result dictionaries.
This is useful for development but weak as a cross-interface contract.

QRMI errors appear as Python exceptions in the QFw driver path. QDMI errors can
originate as QDMI return codes, FoMaC exceptions, or Python exceptions after
binding. The current QFw shim does not normalize those categories.

The unification path is a common error record. It should include category,
code, message, provider code, retryability, call name, device ID, job ID when
available, and raw provider details when safe to return.

</details>

<details>
<summary><strong>Extensibility And Versioning</strong></summary>

The current QFw shim expresses capability with a static descriptor. That is
enough to compare QRMI and QDMI today, but it does not define API versioning or
feature negotiation.

The unification path is category-level versioning. A service should be able to
advertise support for `api_runtime`, `api_data_normalization`,
`api_telemetry`, `api_admission`, and related categories independently. Each
category should define required fields, optional fields, extension points, and
version compatibility rules.

</details>

## QFw Comparison Test Plan

The QFw test path is the practical mechanism for filling this document with
evidence. The current test service is `svc_lib_qpm`. It is started by
`examples/qfw_shim_smoke.sh` with
`examples/qfw_shim_smoke_services.yaml`. The test client is
`examples/tests/test_shim_smoke.py`.

The most useful current command shape is:

```bash
cd /workspace/qfw-container-base/QFw/examples
./qfw_shim_smoke.sh --libs qdmi,qrmi --call get_device_info
```

The `--libs qdmi,qrmi` form runs supported introspection calls once through
each requested lower library. This gives side-by-side output for axes where
both libraries are wired.

### Current One-Call Tests

| Axis | QFw test command | Useful evidence |
|---|---|---|
| Service discovery and descriptor routing | `./qfw_shim_smoke.sh --libs qdmi,qrmi --call test` | Confirms the shim QPM service starts, registers, and is selected by device ID and provider properties. |
| Capability advertisement | `./qfw_shim_smoke.sh --libs qdmi,qrmi --call get_device_info` | The test prints `capability_map` before fan-out. This shows which calls QFw believes QRMI and QDMI can serve. |
| Device introspection | `./qfw_shim_smoke.sh --libs qdmi,qrmi --call get_device_info` | Compare qhw-device records, provider/device IDs, qubit counts, qubit labels, metadata source, and raw/provenance fields. |
| Coupling graph | `./qfw_shim_smoke.sh --libs qdmi,qrmi --call get_coupling_graph` | Compare qhw-coupling records, edges, directionality, supported operations, and loci. |
| Calibration and quality data | `./qfw_shim_smoke.sh --lib qdmi --call get_calibration_snapshot` | Current expected result is a gap or pending binding. This identifies what QDMI can fetch internally but QFw does not yet expose. |
| Backend information | `./qfw_shim_smoke.sh --libs qdmi,qrmi --call get_backend_info` | Current expected result may be pending for both paths. Useful to define what backend metadata should contain. |
| Runtime submission | `./qfw_shim_smoke.sh --lib qrmi --call async_run` | Exercises execution through the current execution owner and validates callback completion behavior. |
| Job metadata | `./qfw_shim_smoke.sh --lib qrmi --call get_last_job_metadata` | Should be run after an execution test or through the default sequence. Captures job metadata behavior after submission. |
| Error model | Run an unsupported combination, such as `./qfw_shim_smoke.sh --lib qdmi --call async_run` | Captures how unsupported calls and lower-library failures are reported today. |

### Default Smoke Sequence

Run the default sequence for the current execution owner:

```bash
cd /workspace/qfw-container-base/QFw/examples
./qfw_shim_smoke.sh --lib qrmi
```

Run a side-by-side introspection comparison:

```bash
cd /workspace/qfw-container-base/QFw/examples
./qfw_shim_smoke.sh --libs qdmi,qrmi
```

The default sequence currently covers service startup, capability-map
inspection when `--libs` is used, selected introspection calls, asynchronous
execution, callback completion, and post-execution metadata.

### Evidence To Capture Per Run

For each axis, the test output should be saved with enough context to support
later comparison. The minimum useful record is:

- QFw branch and commit.
- QRMI and QDMI-on-IQM versions or commits.
- QFw service descriptor used for the run.
- Device ID and provider.
- Command line.
- Full returned payload.
- Whether the payload is raw provider data, FoMaC-extracted data, or qhw-data.
- Error category and traceback when the call fails.

### Gaps In The Current Test Harness

The current tests are sufficient for device and coupling comparison. They are
not yet sufficient for the full interface comparison.

Missing test coverage:

- Admission and admission-control APIs.
- Scheduler-control APIs.
- Telemetry APIs.
- Control-plane authorization.
- QDMI execution through QFw.
- Calibration snapshot normalization through both QRMI and QDMI.
- Result normalization parity between QRMI, QDMI, and native IQM paths.

These gaps should be addressed by adding API categories to `api_qpm` or by
splitting them into smaller service APIs as the QFw design evolves.
