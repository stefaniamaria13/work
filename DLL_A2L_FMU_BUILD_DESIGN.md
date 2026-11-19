# Robust Simulink DLL/A2L Generation Design

## 1. Purpose and fixed build contract

This document defines a safe, modular workflow for preparing a Simulink model and generating a `grt.tlc` DLL together with its matching A2L file.

The generator has one user-facing option only:

```matlab
IncludeIntermediateSignals = false;
```

This option must remain `false` by default. The mandatory generation policy is fixed and must not be exposed as a collection of selectable flags.

The generated build must preserve and describe:

- workspace/model parameters that are actually used by the model;
- required S-Function parameters, including parameters stored in generated structures;
- root-level Inport and Outport interface data;
- intermediate signals only when `IncludeIntermediateSignals` is explicitly `true`.

The generator must not attempt to make every internal signal addressable. In the default build it must not enumerate, rename, or apply `ExportedGlobal` to intermediate block signals.

The output of the process is:

```text
DLL + matching A2L + build log + optional diagnostic build report
```

The optional report is only for diagnostics. It is not a second address database and is not a replacement for the A2L.

## 2. Main safety principle

The preparation script must classify model objects before changing code-generation properties. It must never apply one blanket storage-class operation to all block outputs.

The relevant distinction is:

```text
model object       -> preservation rule       -> generated object       -> A2L entry
parameter          -> parameter rule          -> parameter storage       -> CHARACTERISTIC/MEASUREMENT
root Inport        -> input rule              -> root input storage       -> MEASUREMENT/appropriate object
root Outport       -> output rule             -> root output storage      -> MEASUREMENT/appropriate object
bus                -> hierarchy rule          -> leaves/structure         -> leaf entries where addressable
intermediate leaf  -> debug rule, if enabled  -> exported leaf            -> debug entry
unsupported object -> fail-closed rule        -> no generated guess       -> diagnostic error
```

The final address is a property of the final compiled and linked build. It must not be calculated from a model path, a guessed C name, or MATLAB field order.

## 3. Recommended build flow

The normal build should generate one final DLL/A2L pair. It should not compile a clean model and then compile a prepared model for every request.

```text
1. Validate MATLAB/Simulink/toolchain/licences
2. Create an isolated temporary model copy
3. Run an uninstrumented model update/preflight on the copy
4. Discover and classify mandatory data
5. Discover debug data only when IncludeIntermediateSignals is true
6. Allocate deterministic generated names
7. Apply only category-specific preservation rules
8. Run model update again and validate dimensions/types
9. Generate C code, compile, and link the DLL once
10. Generate the A2L from the final build
11. Validate DLL, A2L, inventory, types, dimensions, and names
12. Write the optional diagnostic report
13. Close models and remove the temporary copy
```

The first model update is a preflight check, not a second complete code-generation build. A separate clean-versus-prepared regression build can be run in CI, but it should not be part of the normal production path unless a project specifically requires it.

Keep the source model unchanged. All preparation must happen on a temporary copy or through reversible model changes protected by guaranteed cleanup logic.

## 4. The only user option

The script should begin with the single setting:

```matlab
IncludeIntermediateSignals = false;
```

### When it is `false`

- discover and preserve mandatory parameters;
- discover and preserve required S-Function parameters;
- discover and preserve root-level inputs and outputs;
- do not enumerate intermediate signals;
- do not apply storage classes to intermediate block outputs;
- do not generate intermediate-signal A2L entries.

### When it is `true`

The script may add addressable intermediate leaf signals for debugging. It must still reject or explicitly report unsupported candidates such as:

- virtual bus aggregate outputs;
- unresolved or variable-size signals;
- signals with unknown data type or dimensions;
- custom objects without a stable generated representation;
- signals whose storage would change model semantics or violate target rules.

Debug mode is therefore an opt-in expansion of the same safe classifier, not a separate blanket-export mode.

## 5. Data discovery and classification

### 5.1 Used workspace and model parameters

Discover parameters from the actual model dependency graph, not from the complete base workspace. The discovery must cover the model workspace, data dictionaries, referenced models where supported, mask/block dialog parameters, and parameters consumed by S-Functions.

For each parameter, record at least:

```text
source scope
model path / owner
logical name
MATLAB/C data type
dimensions
tunable or compile-time status
storage class before preparation
storage class after preparation
whether an addressable generated object is required
```

Do not treat every value used in a Constant block as a runtime calibration parameter. Separate:

- runtime-tunable parameters;
- read-only parameters or measurements;
- compile-time parameters;
- structural parameters such as dimensions, bus definitions, sample times, and variant controls.

If a required parameter is inlined, eliminated, or only represented as a compile-time constant, it has no independent runtime ECU address. The script must either apply the appropriate supported data-object/code-mapping rule or fail with a clear reason. It must not claim that the parameter is addressable merely because it exists in the MATLAB workspace.

Use release-appropriate Simulink data objects, Code Mapping, and storage-class APIs. Do not modify a Constant block output as a substitute for correctly configuring the data object that owns the parameter.

### 5.2 S-Function parameters

S-Function parameters are a mandatory category when they are required by the generated DLL/A2L interface. They must not be handled as ordinary Constant block parameters.

For each S-Function, determine:

- the declared dialog parameters;
- the parameter data type and dimensions;
- whether the parameter is scalar, vector, matrix, structure, pointer, or opaque data;
- whether the generated code stores it directly or inside a generated structure;
- whether the field can be represented by the installed A2L workflow.

If parameters are grouped in a generated structure, keep that structure grouping. Do not flatten fields by guessing offsets from MATLAB field order.

The field address must be resolved from the final generated layout using one of these approaches:

1. symbols or debug/map information from the final build;
2. a small C layout probe compiled with the same compiler, headers, packing, and flags, using `sizeof` and `offsetof`;
3. structure metadata supplied by the installed Simulink/ASAP2 workflow.

The existing padding/offset logic can remain as an implementation detail, but it must be isolated as a layout resolver and validated against the actual compiler layout. Compiler alignment, packing pragmas, type sizes, and build options can all change the result.

Pointer, opaque, dynamically allocated, or variable-size parameters are not automatically addressable A2L parameters. If one is mandatory and no supported representation exists, stop the build and report the exact S-Function and parameter instead of generating a misleading address.

### 5.3 Root-level inputs and outputs

Direction must be determined structurally from the model before any renaming or storage-class operation:

- a root-level `Inport` block is an input;
- a root-level `Outport` block is an output.

The inventory should store the root block path, port index, original/display name, direction, data type, dimensions, and generated name. This direction must not be inferred from the A2L, DLL symbol, or a naming suffix.

The `_Input` and `_Output` suffixes may still be used in generated C/A2L names when they are needed to prevent namespace collisions. They are a naming mechanism only. The source-of-truth for direction is the root port classification.

Nested subsystem ports are local subsystem interfaces. They must not be classified as root-level I/O merely because their block type is `Inport` or `Outport`.

If a root port carries a bus, classify the root port direction first and then apply the bus-leaf policy in Section 6. Do not force the aggregate bus to be a scalar exported signal.

For duplicate display names, allocate a deterministic generated identifier using role, model path, and port index. For example:

```text
root_in__vehicleSpeed__port1
root_out__vehicleSpeed__port1
```

The original name must remain available in the inventory; the generated identifier must be unique after C/A2L sanitization.

### 5.4 Intermediate debug signals

Intermediate signals are not part of the default build. When debug mode is enabled, select only signals that have a stable, addressable representation after code generation.

For an “all intermediate outputs” debug build, recursively inspect the signal hierarchy but preserve only supported leaf objects. A bus parent is not a leaf. A virtual signal with no independent storage is not addressable merely because it appears in the diagram.

## 6. Bus policy and the current failure mode

### 6.1 Why the current bus error is significant

The reported failure says that `Bus Creator1` is being forced to dimension `1`, while `Bus Creator2` input 2 is also seen as a one-element vector. The likely preparation defect is treating a bus aggregate as an ordinary scalar/vector signal and applying a generic signal-preservation rule to it.

This is a diagnosis of the preparation path, not proof that the original model is valid or invalid. The script must compare the uninstrumented preflight copy with the prepared copy and identify the first changed signal or block property. It must not use compiled dimensions from a failed compilation as evidence of the original signal dimensions.

### 6.2 Virtual buses

A virtual bus is a grouping construct. It does not necessarily have independent physical storage. Therefore:

- do not assign `ExportedGlobal` to a virtual bus aggregate;
- do not force a virtual bus aggregate to be scalar;
- do not convert a virtual bus into an array;
- recursively inspect its named elements;
- preserve only addressable leaf data when debug mode is enabled;
- preserve root-bus leaves according to the root port rule when the bus is a mandatory root interface.

Conceptually:

```text
if object is a supported addressable leaf
    apply the category-specific preservation rule
elseif object is a bus
    inspect its children without exporting the aggregate
else
    report unsupported object and fail if it is mandatory
end
```

### 6.3 Nonvirtual buses

A nonvirtual bus has a deliberate generated representation, commonly a structure. Do not convert a virtual bus to nonvirtual simply to make a generic script pass. That changes generated code, memory layout, and potentially model behavior.

If a model already requires a nonvirtual bus, the script must obtain its actual generated structure layout. Each required field must be represented using final type, dimension, and offset information. A parent structure may be retained as a parent object, but field addresses must not be guessed.

### 6.4 Bus versus array

A bus is not a numeric array:

- an array is indexed and has numeric dimensions;
- a bus has named elements;
- a bus element can itself be a vector or matrix;
- an array of buses is a separate construct consisting of repeated bus instances.

Represent bus data hierarchically, for example:

```text
Bus1.signalA          scalar
Bus1.signalB          vector[4]
Bus1.Nested.signalC   scalar
```

Only record array dimensions when the actual Simulink data type is an array/vector/matrix. Only record an array of buses when the model explicitly defines that construct.

See [Simulink signal types](https://www.mathworks.com/help/simulink/ug/signal-types.html) and [arrays of buses](https://www.mathworks.com/help/simulink/ug/combining-buses-into-an-array-of-buses.html).

## 7. Storage-class and naming rules

### 7.1 No blanket instrumentation

Never iterate over every block output and set its storage class. That approach is the direct source of many bus, dimension, virtual-signal, and namespace failures.

Apply preservation only after classification:

```text
parameter       -> parameter/data-object rule
root input      -> root input rule
root output     -> root output rule
debug leaf      -> debug rule, only if enabled
bus aggregate   -> never use the scalar leaf rule
unsupported     -> fail closed
```

### 7.2 Deterministic generated names

Use one name allocator for parameters, root inputs, root outputs, S-Function fields, and optional debug leaves. It must:

- sanitize names to valid C/A2L identifiers;
- prevent collisions between input, output, parameter, and field namespaces;
- preserve a mapping from original model path to generated name;
- produce the same result for the same model and settings;
- detect collisions after sanitization, not only before it.

Do not rename model blocks or alter signal routing solely to create a name. Change only the code-generation-facing data identity that the installed API officially supports.

### 7.3 Avoid semantic changes

The preparation phase must not change:

- bus virtuality;
- signal dimensions;
- data types;
- sample times;
- variant choices;
- block connectivity;
- execution order;
- S-Function parameter values.

If a required storage-class change would change one of these properties, stop and report it for review.

## 8. DLL and A2L generation contract

The DLL and A2L must come from the same final build.

Required sequence:

```text
prepare temporary model
    -> generate source
    -> compile and link DLL
    -> obtain final symbols/layout information
    -> generate A2L
    -> validate the DLL/A2L pair
```

Do not compute final ECU addresses before compilation. Do not reuse an A2L from an older DLL. Build identifiers, target settings, compiler settings, and MATLAB/Simulink release information should be recorded with the artifacts.

For every mandatory item that must be addressable, the final A2L must contain a valid object with matching:

- generated name;
- data type;
- dimensions;
- access semantics;
- final ECU address or supported field address.

Use the release-appropriate Simulink Coder ASAP2 workflow. Current releases document [`coder.asap2.export`](https://www.mathworks.com/help/rtw/ug/a2l-cal-profile.html), while older releases may expose different configuration parameters or generation commands. The script must detect the installed release and use the supported API rather than relying on one hardcoded command for all versions.

For additional context, see [Generate ASAP2 and CDF calibration files](https://www.mathworks.com/help/rtw/ug/asap2-cdf-calibration.html).

## 9. Fail-closed behavior and limitations

The script must stop before final code generation when it cannot safely preserve a mandatory object. At minimum, fail with a diagnostic for:

- unresolved used workspace/model parameter;
- duplicate generated identifier after sanitization;
- unknown data type or dimensions;
- unsupported variable-size mandatory root I/O;
- mandatory virtual bus aggregate selected as an addressable object;
- mandatory S-Function pointer/opaque parameter with no supported adapter;
- unresolvable generated structure layout;
- missing compiler/map/symbol information required for an address;
- inability to generate a matching A2L object.

The script may skip an unsupported optional debug candidate, but it must record the model path and reason. It must never silently skip a mandatory parameter or root I/O.

Important limitations must be visible in the build log:

1. Parameters that are inlined or eliminated do not have independent runtime addresses.
2. Virtual bus aggregates do not necessarily have independent storage.
3. Addresses are build-specific and change when compiler, optimization, layout, or generated code changes.
4. Some S-Function parameter representations cannot be described as ordinary A2L objects without an adapter.
5. `IncludeIntermediateSignals=true` can increase memory, generated symbols, A2L size, and build time.
6. `IncludeIntermediateSignals=false` does not fix a failure in mandatory root-I/O or parameter processing; if the bus error remains, the failure is in the mandatory path or in the original model.

## 10. Validation stages

### 10.1 Before preparation

- verify MATLAB and Simulink release;
- verify Simulink Coder and required support packages;
- verify compiler/toolchain and custom S-Function dependencies;
- verify that the model can be opened and updated;
- capture the model configuration and target settings.

### 10.2 After preflight

- verify that the uninstrumented temporary copy updates successfully;
- record root port types, dimensions, data types, and bus hierarchy;
- record parameter and S-Function dependencies;
- record any existing unresolved model diagnostics.

### 10.3 After preparation and before code generation

- update the prepared model;
- compare mandatory root-port dimensions/types with the preflight inventory;
- compare bus hierarchy and virtual/nonvirtual status;
- verify that no debug-only operation ran when the option is `false`;
- verify that all generated names are unique;
- verify that required parameters still have supported code-generation representation.

### 10.4 After linking and A2L generation

- verify that the DLL exists and matches the current build identifier;
- verify that every mandatory inventory item has exactly one matching A2L object;
- verify A2L data types and dimensions against generated declarations/symbols;
- verify ECU addresses against final symbols/map/sections where available;
- verify structure field offsets using final layout information;
- verify no debug objects appear when `IncludeIntermediateSignals=false`;
- verify optional debug objects are only supported addressable leaves;
- verify no accidental duplicate or unresolved entries;
- retain the complete MATLAB/code-generation/compiler log on failure.

Run model behavior regression tests separately as required by the project. The normal generation path should not compile a second clean artifact just to produce the final DLL/A2L pair.

## 11. Modular script organization

The implementation should be split into small functions with one responsibility each. A practical layout is:

```text
buildDllA2l.m
  settings = defaultSettings();
  environment = validateEnvironment(settings);
  buildDir = createTemporaryBuildCopy(modelPath);
  cleanup = onCleanup(@() cleanupBuild(buildDir));
  preflight = runPreflight(buildDir, settings);
  inventory = discoverMandatoryData(buildDir, preflight, settings);
  inventory = discoverDebugData(buildDir, preflight, inventory, settings);
  inventory = allocateGeneratedNames(inventory);
  applyPreparation(buildDir, inventory, settings);
  validatePreparedModel(buildDir, preflight, inventory, settings);
  artifacts = buildDllOnce(buildDir, settings);
  artifacts.a2l = generateA2lFromFinalBuild(buildDir, artifacts, inventory, settings);
  validateArtifacts(artifacts, inventory, settings);
  writeBuildReport(artifacts, inventory, settings);
```

The only setting exposed at the beginning of the top-level script should be:

```matlab
IncludeIntermediateSignals = false;
```

Release-specific API differences should be isolated inside adapter functions. The classifier and validation rules should not be duplicated across MATLAB releases.

## 12. Diagnostic build report

The generator may write a small report such as `build_report.json` or a MATLAB structure converted to JSON. It should contain diagnostics, not a duplicate address catalog:

```text
build identifier
model name and source fingerprint
MATLAB/Simulink release
target/compiler information
IncludeIntermediateSignals value
mandatory parameter count
S-Function parameter count
root input/output counts
debug candidate/preserved/skipped counts
generated-name mappings
skipped items and reasons
validation results
```

Final addresses remain owned by the final A2L/DLL build artifacts. The report should not become a required input to code generation.

## 13. Remote licensed build and local Python environment

A local Python virtual environment can orchestrate the build, but it cannot replace the MATLAB/Simulink Coder license or compiler environment.

The local environment can:

- validate the request and model package;
- submit a job to the internal licensed worker;
- poll status and retrieve logs;
- validate checksums and artifact identity;
- inspect the build report and A2L consistency;
- package the final artifacts.

The remote worker must provide the pinned MATLAB/Simulink release, Simulink Coder, compiler, custom libraries/S-Functions, support packages, and license access. It can run a non-interactive MATLAB entry point such as `matlab -batch`, subject to the organization’s license setup. See [Build models from the command line](https://www.mathworks.com/help/rtw/ug/building-models-from-the-dos-window-command-line.html) and [MATLAB startup options](https://www.mathworks.com/help/matlab/matlab_env/startup-options.html).

MATLAB Compiler is not a substitute for the licensed Simulink Coder build environment; MathWorks lists Simulink Coder as unsupported for this packaging scenario. See [MathWorks Compiler support](https://www.mathworks.com/products/compiler/compiler_support.html).

The returned artifact package should contain:

```text
DLL
A2L
optional build report
build identifier/fingerprint
build log
MATLAB/release/compiler information
```

## 14. Final recommendation

The robust implementation is a single category-aware generator with one optional debug switch:

1. Keep `IncludeIntermediateSignals = false` by default.
2. Always discover used workspace/model parameters, S-Function parameters, and root-level I/O.
3. Determine root direction from root `Inport`/`Outport` blocks before renaming.
4. Apply storage-class rules only to supported classified data objects or addressable leaves.
5. Never export a virtual or aggregate bus as though it were a scalar signal.
6. Resolve S-Function structure fields from the final compiler layout.
7. Generate the A2L only after the final DLL/link information exists.
8. Validate every mandatory item and fail closed on ambiguity.
9. Generate one final build from an isolated temporary copy.
10. Keep optional diagnostics in a report, not in a second address representation.

This approach removes the current source of bus surprises, keeps the default DLL/A2L compact, preserves the required model interface, and makes debug visibility an explicit opt-in without turning every model signal into exported data.
