# Canonical Circuit Form: OpenQASM3 vs QIR

This note compares OpenQASM3 and QIR as the **canonical circuit form** carried by
the QFw front-end contract for the execution facet
(`qpu-frontend-contract` §12.1). It supports the *Program Representation And
Placement* axis of [`qrmi-qdmi-analysis.md`](./qrmi-qdmi-analysis.md).

The contract carries one canonical circuit form. Each driver transcodes that
form into whatever its library actually ingests, and the per-resource descriptor
declares the ingestion format. The comparison below is grounded in what QRMI and
QDMI-on-IQM actually accept on the IQM q20 — which is the constraint that drives
the decision, because neither library ingests a neutral source language
directly.

## What the libraries actually ingest (verified)

| Library (IQM) | Accepted submission format(s) | Neutral? |
|---|---|---|
| **QRMI** | IQM run-request **JSON** only (`Payload.IQMServer(iqmjson=...)`). Its Qiskit adapter converts Qiskit circuits → IQM JSON above the core API. | No — provider-native |
| **QDMI-on-IQM** | `QDMI_PROGRAM_FORMAT_QIRBASESTRING` (QIR base, string) and `QDMI_PROGRAM_FORMAT_IQMJSON` (+ `CALIBRATION` for calibration jobs). **No QASM.** | QIR is neutral; IQM JSON is not |

QDMI's format *enum* defines QASM2, QASM3, QIR base/adaptive, QPY, and IQM_JSON —
but the IQM device advertises only `QIRBASESTRING` + `IQMJSON`
(`SUPPORTED_PROGRAM_FORMATS` in `iqm_device.cpp`). So on this device:

- **Neither library takes QASM.** A QASM canonical form is always transcoded.
- **The one format both accept is IQM JSON** — but it is provider-specific, so it
  cannot be the neutral canonical form (it would defeat portability).
- **QIR-base is accepted natively by QDMI, but not by QRMI** (which needs IQM
  JSON).

### Transcode matrix (canonical → each library's input)

| Canonical form | → QRMI (needs IQM JSON) | → QDMI (needs QIR-base or IQM JSON) |
|---|---|---|
| **OpenQASM3** | QASM3 → Qiskit → IQM JSON. Mature (qiskit + qiskit-iqm/iqm-client). | QASM3 → IQM JSON, submit as `IQMJSON`. Same transcoder as QRMI. |
| **QIR-base** | QIR → circuit → IQM JSON. Immature for IQM (no turnkey QIR→IQM path). | QIR-base submitted ~directly (native). QDMI compiles QIR→IQM internally. |

The important asymmetry: with **OpenQASM3**, both drivers can converge on **one
transcoder** (QASM3 → IQM JSON) and simply submit it two ways (QRMI
`Payload.IQMServer`; QDMI job parameter with `IQMJSON`). With **QIR**, QDMI gets a
neutral, near-passthrough path but QRMI needs an immature QIR→IQM-JSON step.

## OpenQASM3

- **Source-level, human-readable** circuit language; easy to inspect, diff, and
  debug in tests (the shim smoke test already sends QASM — currently 2.0).
- **Large ecosystem**: Qiskit parses/emits QASM3; qiskit-iqm/iqm-client transpile
  Qiskit → IQM. So QASM3 → IQM JSON is a well-trodden path.
- **Dynamic features** (mid-circuit measurement, classical control, real-time)
  are expressible in the language.
- **Not ingested by either library here** — always requires transcoding. For this
  device that transcode target is IQM JSON for both.

## QIR

- **Compiled LLVM-based IR**, not human-readable; designed as a neutral compiler
  interchange (QIR Alliance). Two profiles: **base** (static) and **adaptive**
  (dynamic control flow).
- **QDMI's neutral submission path**: QDMI-on-IQM accepts `QIRBASESTRING` natively
  and compiles it to IQM internally. This keeps the QDMI submission boundary
  vendor-neutral end-to-end — QRMI has no equivalent neutral format for IQM.
- **QRMI side is the cost**: no turnkey QIR→IQM-JSON path today; would need a QIR
  reader → gate list → IQM JSON.
- **Adaptive limitation here**: QDMI-on-IQM supports QIR **base** only, not
  adaptive — so dynamic circuits are not ingestible on this device regardless.
- Harder to debug/inspect in tests; larger, opaque payloads.

## Axis comparison

| Axis | OpenQASM3 | QIR |
|---|---|---|
| Neutrality of the canonical form | Neutral source language | Neutral compiled IR |
| Ingested directly by QRMI (IQM) | No (→ IQM JSON) | No (→ IQM JSON, immature) |
| Ingested directly by QDMI (IQM) | No (→ IQM JSON via transcode) | **Yes** (`QIRBASESTRING`) |
| Keeps QDMI submission neutral | No (we hand it IQM JSON) | **Yes** (neutral at the boundary) |
| Transcode tooling maturity (this stack) | **High** (Qiskit + qiskit-iqm) | Low for QRMI/IQM |
| Shared transcoder across both drivers | **Yes** (one QASM3→IQM-JSON) | No (QIR for QDMI, IQM-JSON for QRMI) |
| Human-readable / debuggable | **Yes** | No |
| Dynamic / adaptive circuits | Expressible in language | Adaptive profile exists, but **not supported by QDMI-on-IQM** |
| Ecosystem breadth | Broad (source interchange) | Growing (compiler IR) |

## Recommendation

**Adopt OpenQASM3 as the canonical circuit form for the near-term execution
milestone (IQM q20), and design the contract to carry a declared format so QIR
can be adopted later without a contract break.**

Rationale:

1. **One transcoder serves both drivers today.** QASM3 → IQM JSON is mature
   (Qiskit + qiskit-iqm), and both QRMI (`Payload.IQMServer`) and QDMI-on-IQM
   (`IQMJSON`) accept IQM JSON. QIR would give QDMI a near-passthrough but leave
   QRMI on an immature QIR→IQM path.
2. **Debuggability during bring-up.** A readable canonical form matters while the
   execution facet is new; the smoke test is already QASM-based.
3. **No functional loss on this device.** QDMI-on-IQM supports only QIR *base*
   (static), so QIR buys no dynamic-circuit capability here.

Two caveats worth stating to keep the spec honest:

- **QIR is the strategically neutral submission path**, and its advantage is real
  but currently unused: it lets QDMI stay vendor-neutral at the submission
  boundary, whereas transcoding to IQM JSON is an IQM-specific shortcut. This is
  itself a QRMI/QDMI difference for the *Program Representation And Placement*
  axis — QDMI accepts a neutral compiled IR; QRMI accepts only provider-native
  JSON.
- **Don't hardcode a single form forever.** Mirror QDMI's model: the contract
  carries an explicit program-format tag, the descriptor declares each library's
  supported ingestion formats (as QDMI already exposes via
  `supported_program_formats()`), and the front-end transcodes. That makes
  "OpenQASM3 today, QIR-base where a device prefers it" a configuration choice,
  and lets a future adaptive-capable device negotiate QIR-adaptive without
  breaking the contract.

**Net:** default canonical = **OpenQASM3**; build the transcode layer to emit IQM
JSON for both drivers now; keep the format declared (not implicit) so QIR remains
a first-class future option — especially as the neutral QDMI submission path and
for adaptive circuits once devices support them.
