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

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

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

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

</details>

## Device Introspection

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

</details>

## Calibration And Quality Data

<details>
<summary><strong>QRMI Behavior</strong></summary>

The QRMI path carries the provider-native IQM calibration payload through the
interface boundary. In QFw, the QRMI shim driver calls
`self._qpu().target().value`, parses that JSON once, and caches it in
`QFw/services/svc_lib_qpm/drivers/qrmi_driver.py:175-186`.
`get_calibration_snapshot()` then passes the QRMI target sections directly into
`qhw_iqm.normalize_calibration`: `dynamic_quantum_architecture` becomes
`dynamic_architecture`, `calibration_set` is passed through as `calibration_set`,
and `quality_metrics` is passed through as `quality_metric_set`
(`QFw/services/svc_lib_qpm/drivers/qrmi_driver.py:212-226`).

The underlying QRMI Python binding maps
`QuantumResource(..., ResourceType.IQMServer)` to the Rust `IQMServer`
implementation (`qrmi/src/pyext.rs:102-107`) and exposes `.target()` by calling
`self.qrmi.target().await` (`qrmi/src/pyext.rs:226-231`). The IQM server
implementation builds one JSON object containing the full responses from:

- `get_dynamic_quantum_architecture_v1`, stored under
  `dynamic_quantum_architecture` (`qrmi/src/iqm/server.rs:236-251`)
- `get_calibration_set_v1`, stored under `calibration_set`
  (`qrmi/src/iqm/server.rs:253-268`)
- `get_quality_metrics_v1`, stored under `quality_metrics`
  (`qrmi/src/iqm/server.rs:270-285`)

Those generated IQM client calls hit the direct calibration-set endpoints:
`/api/v1/calibration-sets/{qc}/{cal_set}`,
`/api/v1/calibration-sets/{qc}/{cal_set}/dynamic-quantum-architecture`, and
`/api/v1/calibration-sets/{qc}/{cal_set}/metrics`
(`qrmi/dependencies/iqm_client/src/apis/calibration_sets_api.rs:66-71`,
`qrmi/dependencies/iqm_client/src/apis/calibration_sets_api.rs:112-117`, and
`qrmi/dependencies/iqm_client/src/apis/calibration_sets_api.rs:159-164`).

After QRMI returns the target JSON, `qhw-iqm` normalizes it without doing the
QDMI-style metric down-select. `normalize_calibration()` reads the full
`calibration_set` and `quality_metric_set` objects, counts their observations,
and places the IQM observation sets under `extensions["iqm.v1"]`
(`qhw-iqm/src/qhw_iqm/normalize.py:138-157` and
`qhw-iqm/src/qhw_iqm/normalize.py:174-219`). `_iqm_observation_set()` preserves
the observation-set identity fields and the full `observations` arrays
(`qhw-iqm/src/qhw_iqm/normalize.py:444-460`). At the raw QRMI `target()` layer,
the complete endpoint JSON objects are available; at the qhw-normalized layer,
the full calibration and quality observation arrays remain available for
downstream analysis.

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

The QDMI path does not pass the full IQM observation sets through Python. QFw
opens the IQM QDMI shared library through MQT Core's FoMaC loader in
`QFw/services/svc_lib_qpm/drivers/qdmi_driver.py:89-119` and serves
`get_calibration_snapshot()` by calling
`fomac_normalize.extract_calibration(self._device())` in
`QFw/services/svc_lib_qpm/drivers/qdmi_driver.py:145-152`. The QFw extractor
can only ask the FoMaC/QDMI object for standardized accessors: per-site
`t1()`/`t2()`, per-operation `fidelity()`/`duration()`, and the device duration
unit (`QFw/services/svc_lib_qpm/drivers/fomac_normalize.py:47-78` and
`QFw/services/svc_lib_qpm/drivers/fomac_normalize.py:243-271`). The normalized
record then stores only `qubit_metrics`, `gate_metrics`, and `duration_unit`
under `extensions["qdmi.fomac.v1"]`
(`QFw/services/svc_lib_qpm/drivers/fomac_normalize.py:123-164`).

The down-select happens inside QDMI-on-IQM before QFw sees the data. The IQM
device session stores only site identity plus `t1_`/`t2_` and operation fidelity
maps (`QDMI-on-IQM/src/iqm_device.cpp:104-137` and
`QDMI-on-IQM/src/iqm_device.cpp:205-210`). QDMI-on-IQM does fetch the IQM
quality-metrics endpoint (`QDMI-on-IQM/src/internal/iqm_api_config.cpp:47-50`
and `QDMI-on-IQM/src/iqm_device.cpp:460-470`), but it immediately flattens
valid observations into a `dut_field -> value` map
(`QDMI-on-IQM/src/iqm_device.cpp:479-488`) and extracts only:

- `characterization.model.<qubit>.t1_time` into `site->t1_`
  (`QDMI-on-IQM/src/iqm_device.cpp:490-496`)
- `characterization.model.<qubit>.t2_time` into `site->t2_`
  (`QDMI-on-IQM/src/iqm_device.cpp:497-502`)
- `metrics.ssro.measure.<impl>.<qubit>.fidelity`
  (`QDMI-on-IQM/src/iqm_device.cpp:508-521`)
- `metrics.rb.prx.<impl>.<qubit>.fidelity:par=d2`
  (`QDMI-on-IQM/src/iqm_device.cpp:524-538`)
- `metrics.irb.cz.<impl>.<q1>__<q2>.fidelity:par=d2`
  (`QDMI-on-IQM/src/iqm_device.cpp:541-557`)

The public query functions then expose only the implemented QDMI properties:
site index/name/T1/T2 and operation name/qubit count/parameter count/sites plus
fidelity when a mapped fidelity exists
(`QDMI-on-IQM/src/iqm_device.cpp:1659-1685` and
`QDMI-on-IQM/src/iqm_device.cpp:1688-1791`). Although QDMI has an operation
duration property in the standard, QDMI-on-IQM documents that IQM operation
durations are not currently supported (`QDMI-on-IQM/docs/usage.md:194-201`).

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

QRMI currently acts as a provider-native data transport for IQM target data:
downstream code can receive the dynamic architecture, calibration set, and full
quality-metric observation arrays. QDMI-on-IQM currently acts as a standardized
property projection: it fetches the IQM quality-metrics response but exposes
only the fields it maps into QDMI properties. In the current QFw/QDMI path,
Python sees T1/T2 and selected measure/prx/cz fidelities, not the original IQM
observation objects.

That means the statement "a script that needs to analyze calibration data in
detail currently cannot do that with QDMI" is true for the code inspected here.
Such a script can use QDMI for the mapped metrics: per-qubit T1/T2 and selected
gate fidelities. It cannot use QDMI to inspect the full IQM calibration data set:
unmapped `dut_field` entries, readout error components such as
`error_0_to_1`/`error_1_to_0`, `t2_echo_time`, observation IDs, timestamps,
units, uncertainty, invalid flags, and other provider-native observation fields
are not available after QDMI-on-IQM's down-select. They are dropped before the
QFw FoMaC adapter builds its `qdmi.fomac.v1` record.

This limits applications that need deeper calibration analysis. Any analysis
that depends on the complete IQM observation set must currently use the QRMI
path, the native IQM path, or a new QDMI extension that exposes raw calibration
and quality-metric observation sets. The current QDMI path is appropriate for
portable device-property queries, but not for detailed provider-native
calibration analytics.

</details>

## Telemetry

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

</details>

## Device Authentication

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

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

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

</details>

## Program Representation And Placement

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

</details>

## Error Model

<details>
<summary><strong>QRMI Behavior</strong></summary>

</details>

<details>
<summary><strong>QDMI Behavior</strong></summary>

</details>

<details>
<summary><strong>Comparison Analysis</strong></summary>

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
