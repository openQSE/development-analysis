# QRMI And QDMI Resolution

This document records the resolution of the QRMI and QDMI comparison. The
comparison itself lives in [`qrmi-qdmi-analysis.md`](./qrmi-qdmi-analysis.md),
which examines the two interfaces across sixteen axes with the QFw shim as the
test vehicle. Every observation cited here is drawn from that document and
carries its version pins. This document asks what should happen next, given
what was observed and given the position the two projects are actually in.

The comparison keeps reaching the same structural finding. QDMI is a device
contract implemented by providers. QRMI is a resource interface consumed by
schedulers and middleware. They are not two candidates for one slot. One sits
above where the other sits.

The natural reading of that finding is composition, a stack that deploys them
as the layers they are, one on the other. This document declines to recommend
that reading. The two projects are developed by organizations in competition.
Each library anchors its sponsor's ecosystem, and composition asks one sponsor
to build its product on the other's contract. Nothing in either project's
direction suggests that will happen. The rest of this document takes continued
separation as given and records the avenues that remain open, which turn out
to be most of them.

## The Layering Finding Moves Into The Specification

The phrase "different layers" does not have to describe a deployment. It can
describe the shape of the specification. A specification can be written as two
contracts. A device contract states what a device is and can do. A resource
contract states how work is admitted, placed, accounted, and reported against
a fleet of devices. The two contracts can be conformed to independently.

Each project then certifies against the layer or layers it claims. QDMI is
already the shape of the device contract. QRMI's sponsor can conform its
resource surface without ever loading a QDMI device library. Neither adopts
the other's code. Portable middleware targets the contracts rather than either
library.

The Interface Role And Scope axis of the comparison said the first decision
for a common specification is which layer is being specified. Under this
avenue that decision stops being a prerequisite for cooperation and becomes
the organizing principle of the specification itself. This is the separate but
conforming outcome, and it is the primary recommendation here.

## Standardize The Artifacts Where The APIs Cannot Converge

A weaker and more achievable standardization runs through the payloads rather
than the calls. The pieces already exist in the comparison. Runtime Submission
argues for a qtask envelope separated from an exchange payload. Data
Normalization concludes that the record schema is the contract and the
normalizer is a per-source adapter. The companion document
[`canonical-circuit-form-qasm3-vs-qir.md`](./canonical-circuit-form-qasm3-vs-qir.md)
examines the circuit form that would cross the boundary.

Competing runtimes sharing common artifact formats is the oldest
interoperability pattern there is, and it asks the least of the sponsors
because neither cedes API design. HTTP is the familiar example. Rival
browsers and rival servers compete on everything else, yet they all exchange
the same requests and the same HTML, and the web holds together because the
things exchanged are standard even though the programs exchanging them never
merged. The `qhw-*` records already demonstrate the mechanics. Two adapters, one fed raw provider data and one fed typed neutral
properties, converged on one record that carried identical counts from live
hardware. Even if verb-level conformance never happens, artifact
standardization delivers most of what a portable application actually needs.

## The Adapter Layer Requires No Agreement

If neither sponsor conforms to anything, the integration point moves to the
consumer, meaning middleware and sites. The QFw shim is the existence proof
that the consumer can hold that position, and the comparison has priced it.
The price is itemized across the axes. It includes a hand-maintained
capability map that drifted silently until a routing assertion caught it, two
normalization adapters, a serialization responsibility that differs per path,
and coupling to a provider SDK's private surface that broke on an upgrade.

None of that is prohibitive, and all of it recurs per consumer. The recurrence
is the argument for pushing as much of it as possible into a shared
specification. It is also the guarantee that no political outcome can strand
the users. Portability is purchasable without either sponsor's agreement.

## Procurement Turns A Specification Into Behaviour

Specifications in this space are not adopted because vendors agree with each
other. They are adopted because buyers require them. MPI is the precedent.
Vendor implementations conform because acquisition documents said they must.
The buyers here are the HPC sites deploying quantum resources, and openQSE is
positioned to speak for them. Conformance language in acquisition documents,
of the form "implements the device contract, verified by the conformance
suite", converts the specification from an invitation into a requirement.

The economics favour it. The sponsors differentiate on hardware quality,
compilation, and runtime services, not on the shape of a status enumeration or
a submission verb. Conforming at the interface therefore costs them little,
and refusing it costs them procurements. That asymmetry is what made vendor
conformance to MPI sustainable among competitors, and there is no obvious
reason it fails here.

## What A Specification Must Avoid

A specification does not become real by being published. It becomes real by
being implemented. If neither project implements it, the specification does
not replace the two libraries. It becomes a third thing occupying the same
ground. Users who had to choose between two interfaces now have to weigh
three. Portable code written against the unimplemented specification runs on
nothing real, so someone still has to write and maintain adapters from the
specification to each library. That is the original problem with one more
layer added. Two guards keep this from happening.

The first is governance neutral enough that both sponsors can participate
without adopting each other's work. The two-contract split provides that
structurally, since each sponsor can engage with only the layer it cares
about.

The second is derivation by subtraction. The specification should be assembled
from the observed intersection and the recorded divergences of implementations
that exist, which is the material of the comparison, rather than designed
fresh for both projects to ignore. Subtraction is the starting discipline,
not a ceiling. Once the specification is grounded in what exists, later
revisions take their direction from the users above it. The conformance suite
is what makes the difference enforceable. Without one, conformance is a press
release.

## What The Comparison Contributes Under Every Outcome

Whether the future is conformance, artifact standardization, consumer-side
adapters, or continued divergence, the specification work needs the same
inputs. Those inputs are the places where the two interfaces disagree in ways
a portable caller can observe. The comparison records them per axis, and four
recur often enough to state as themes.

Support must be discoverable before it is relied on. The absent capability
advertisement, the `acquire()` that returns a token it did not earn, and the
monolithic trait that cannot express partial implementation are one finding in
three places.

What the provider already supplies must survive the interface. The discarded
job timeline, the down-selected calibration observations, and the unreturned
run request are one finding in three places.

The envelope must be separate from the payload. The granularity split between
a whole run request and a bare circuit under the same format name, and the
format declaration one interface has nowhere to put, are the same finding
twice.

Failure must propagate as failure. The null-substituted target document that
resurfaced two layers up as a device-data fault is the cautionary instance.

The table below gathers the per-axis conclusions and restates each one as an
obligation a two-contract specification would need to discharge.

| Axis | What a specification must pin down |
|---|---|
| Interface role and scope | The specification should be written as two separate contracts. The device contract describes what a device is and can do. The resource contract describes how work is admitted, placed, accounted, and reported. The content of both contracts should be driven first by what the two libraries offer today, since that is the behaviour that exists and has been measured. It should then be driven by the upper-layer users, meaning the applications, schedulers, and middleware above the contracts, and by what those users ultimately need. Starting from the libraries keeps the specification grounded. Handing direction to the users keeps it from being permanently limited to what the libraries happen to do now. Every requirement that follows should state which of the two contracts it belongs to, so that neither contract quietly grows into the other's territory. |
| Device discovery and capability advertisement | Software should learn what devices exist by asking through the interface, not by holding site configuration. At the same time, the resource manager must be able to limit what that query returns, so a batch job sees only the devices its scheduler granted. Each resource must also be able to answer what it supports, including which program formats it accepts. When a capability is absent, the interface returns an explicit not-supported answer instead of empty data, so a caller can tell missing support from missing information. |
| Admission | The specification must define exactly what an admission call such as acquire promises, and a caller must be able to ask whether a given resource actually implements that promise. The comparison found one acquire call with three different meanings across backends. When a backend cannot actually reserve or hold a device, the call must say so by failing, not succeed by returning a token that holds nothing. |
| Admission control configuration | The specification should not grow its own policy engine for quotas, rate limits, or quality of service. Sites already express that policy in their scheduler. What the interface must supply is the information the scheduler needs to make those decisions device-aware: current load, queue depth, an expected wait estimate, a vocabulary of device states, and enough accounting identity to reconcile usage afterwards. |
| Device scheduler control | A site scheduler orders work, and today the provider's own queue reorders it with no channel between the two. Submission must therefore carry the site's scheduling intent, at minimum a priority or ordering key, a deadline, and a reference to any reservation the site holds. The specification must define what the provider does when it cannot honour that intent, and refusing the job must be one of the allowed answers, so the site's decision is never silently overridden. |
| Runtime submission | A submission has two parts that must be kept separate. The envelope carries execution options such as shot count, placement, and result routing. The payload carries the compiled program itself. Which program formats qualify as portable exchange formats should be decided against agreed criteria, not inherited from whichever format a provider library already accepts. A caller that needs provider-specific behaviour can still submit a provider-native payload, but doing so is an explicit choice that gives up portability for that call. |
| Job lifecycle and results | Both interfaces already return the provider's job identifier, so the specification should simply standardize that. What needs to be added is provenance. A caller must be able to retrieve, from the job itself, what was actually submitted, with which calibration set and which execution options. That must hold even when a lower layer assembled the final request, because usage reconciliation and incident forensics need more than the job id. |
| Device introspection | Device information should be exposed as typed, vendor-neutral properties in the QDMI style, because that is what portable tools can consume directly. But the neutral model must reserve a defined place for provider identity and provenance, such as the identifier of the calibration set the data came from. Today that identity survives only in the raw provider document, so choosing the portable model means losing it. |
| Calibration and quality data | A neutral projection of calibration data, such as T1, T2, and gate fidelities, serves most consumers and should stay. But the provider's full observation set is larger than any projection, and the comparison found the device implementation silently discarding whatever it did not map. The specification therefore needs a defined channel through which the complete provider observation set can be requested, so calibration-analysis workloads do not have to abandon the portable interface entirely. |
| Telemetry | The provider already reports how long a job spent queued, compiling, and executing, and both interfaces discard it. The specification must require that timing to pass through in a defined shape rather than invent a new timing model. Queue depth and device load must be queryable, because admission and scheduling decisions have nothing to act on without them. And because the cost of observing is part of the contract, expectations about transport behaviour such as connection reuse, and about what is cached and for how long, belong in the specification rather than being left to each implementation. |
| Device authentication | How a credential is obtained must stay separate from the calls that use it, which both interfaces already get right. The injection contract must support both deployment styles observed: credentials delivered through the environment by a scheduler prologue, and credentials passed as parameters by an in-process client. |
| Control-plane authorization | Neither interface has privileged operations today, and one execution credential is enough. That changes the moment any operator-facing call is added, such as setting a limit or injecting a credential. If the specification adds such a surface, it must split authorization at the same time: the protected calls named explicitly, operator privilege distinct from a user's right to execute, and a defined answer when an unprivileged caller attempts a protected call. |
| Data normalization | Neither interface emits the record a consumer ultimately wants, so the specification should standardize the record schema itself and treat each normalizer as a per-source adapter. The schema must mark provider-specific fields as explicitly optional, so a source that returns rich raw data and a source that returns a thin neutral projection can both fill the same record honestly, without either inventing data it does not have. |
| Program representation and placement | The format of a submitted program must be declared alongside the program bytes, so a device can advertise the formats it accepts and reject the ones it does not. Today one interface has no place to express that at all. Placement must likewise become an explicit envelope field with defined meaning. In both interfaces today, placement exists only as physical qubit names embedded inside the serialized program, which a runtime cannot validate, retarget, or reliably read. |
| Error model | Every failure must reach the caller in a defined form, either a typed error code or a raised error. Substituting empty data for a failed fetch is prohibited, because the comparison watched a connectivity failure travel two layers up and present as a device-data fault. Diagnostics must also work when the interface is embedded as a library, since a logging facility that only functions in standalone binaries leaves the embedded case blind. |
| Extensibility and versioning | The interface should grow by adding property values behind stable function signatures, so an implementation built against an older version keeps working. Partial implementations must be expressible through a typed not-supported answer rather than through no-op methods. A caller must be able to ask an implementation which version it speaks. Vendors need reserved space for their own extensions, but whatever accumulates in that space should be read as a list of what the specification is missing and folded into the next revision. |

Two artifacts of the comparison are worth carrying directly into that work.
The first is the QFw shim's capability map, which records the calls each
library serves for each device and a preference to break ties. Today it is
hand-maintained knowledge that drifts. Under a specification it becomes
something an implementation declares and a test suite verifies, which makes it
the seed of the conformance declaration. The second is the pair of measurement
scripts, which stand in the same relation to a performance annex. Cold-start
cost, warm-path cost, and connection behaviour are already being compared
across implementations of the same contract, which is exactly what a
conformance suite would do on purpose.

The resolution of this comparison is therefore not a verdict between the
libraries, and it does not depend on their sponsors reconciling. It is the
conversion of observed divergence into specification obligations. The result
is two contracts, independently conformable, derived from what was measured
rather than from what was hoped.
