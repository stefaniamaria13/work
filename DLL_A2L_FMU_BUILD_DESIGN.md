# Robust DLL/A2L/FMU Build Design

This document defines a stable workflow for preparing a complex Simulink model, generating a `grt.tlc` DLL and A2L file, and then allowing a later FMU wrapper step to activate or deactivate model data.

It is based on the workflow described in [AGENT.md](./AGENT.md), especially the distinction between the original FMI Kit route and the newer `DLL_build` route.

## 1. The required behavior

The final FMU wrapper has options that can activate or deactivate model data and also provides XCP functionality. Therefore the DLL/A2L build must make the data addressable before the wrapper runs.

The intended data scope is:

- workspace parameters used by the model;
- S-Function parameters used by the model;
- root-level inputs and outputs;
- intermediate signals only when debug/measurement support is requested;
- no requirement to expose every internal signal in normal operation.

The key constraint is:

> The wrapper can select an already-preserved variable, but it cannot create a valid ECU address for a signal that was optimized away during code generation.

Therefore, the build must preserve a configurable superset of data, while the wrapper selects the active subset later.

## 2. Recommended high-level architecture

```text
Original model and libraries
        |
        v
Create isolated temporary build copy
        |
        v
Preflight/update the unmodified copy
        |
        v
Discover parameters, root I/O, and optional debug signals
        |
        v
Apply category-aware preservation rules
        |
        v
Generate code with grt.tlc
        |
        v
Compile and link the DLL
        |
        v
Generate A2L from the final generated code/link information
        |
        v
Generate a data registry/TQA from the same build
        |
        v
FMU wrapper selects active variables and creates modelDescription.xml
```

The preparation stage and the wrapper stage should have different responsibilities:

| Stage | Responsibility |
|---|---|
| Model preparation | Make selected data survive code generation and have stable generated-code representation |
| `grt.tlc` build | Generate the C source, compile it, and link the DLL |
| A2L generation | Describe the final linked ECU data and addresses |
| Data registry/TQA | Provide a machine-readable index shared by the DLL, A2L, and wrapper |
| FMU wrapper | Decide which available variables are active in this particular FMU |

## 3. Do not build the clean model and prepared model twice by default

The normal production workflow should not be:

```text
build clean model
build prepared model
```

That doubles the expensive code-generation and compilation work and creates two artifacts that are not needed by the final FMU.

Instead:

1. Keep the original model unchanged on disk.
2. Create or load a temporary prepared copy.
3. Run a normal model update/preflight on the copy before adding instrumentation.
4. Apply the preparation rules to that copy.
5. Generate the DLL and A2L once.
6. Close and delete the temporary copy.

The preflight update is useful because it verifies the original topology before instrumentation, but it is not a second full code-generation build.

A separate clean-versus-prepared regression build can be run periodically or in CI, but it does not need to run for every user FMU build.

## 4. Build profiles

Use explicit profiles instead of one flag that performs several unrelated actions.

Recommended configuration:

```text
PreserveParameters       = true
PreserveRootIO           = true
PreserveDebugSignals     = true or false
IncludeDebugSignalsInFMU = false by default
GenerateXCPAddressData   = true
```

The distinction is important:

- `PreserveDebugSignals` is a code-generation decision. It determines whether intermediate signals remain available in the DLL.
- `IncludeDebugSignalsInFMU` is a wrapper decision. It determines whether those already-preserved signals appear in this FMU's interface.

If intermediate signals must be activatable after the DLL/A2L build, `PreserveDebugSignals` must be enabled during the DLL build even when `IncludeDebugSignalsInFMU` is initially false.

If debug signals are not needed for a particular DLL, disabling `PreserveDebugSignals` saves code size, memory, build time, and runtime overhead. They cannot then be added later without rebuilding the DLL.

## 5. Data classification rules

Do not apply one blanket `StorageClass` operation to every block output. First classify each candidate.

### 5.1 Workspace parameters

Discover variables that are actually used by the model, including variables from:

- the base workspace;
- the model workspace;
- a data dictionary;
- referenced models, where applicable;
- mask and block dialog parameters;
- S-Function parameter lists.

Do not export every variable in the base workspace. Export only variables that are model dependencies and are meaningful to the wrapper/XCP interface.

Separate parameters into:

- tunable runtime parameters;
- read-only calibration/measurement parameters;
- compile-time parameters;
- structural parameters such as dimensions, bus types, sample times, and variant controls.

Compile-time and structural parameters should not automatically be presented as runtime-tunable XCP characteristics. Changing them may require rebuilding the model or may invalidate the generated interface.

For parameters that must remain visible and addressable, use explicit parameter objects and code-generation settings appropriate for the installed MATLAB release. A storage class other than `Auto` can prevent parameter data from being inlined or eliminated, but it also changes the generated-code interface.

### 5.2 S-Function parameters

Treat S-Function parameters as a separate category rather than assuming they behave like Constant block values.

For each S-Function parameter, record:

- the S-Function block path;
- parameter index and logical name;
- scalar/vector/matrix/structure type;
- dimensions;
- tunable or read-only status;
- whether the value is stored directly, inside a generated structure, or behind a custom interface;
- the A2L data type and conversion method.

If an S-Function parameter is a pointer, opaque object, or custom structure, it may not be directly representable as a normal A2L characteristic. Such parameters need an explicit adapter or a documented exclusion.

Do not infer S-Function parameter addresses from assumed MATLAB structure padding. Use the final generated symbol/map information or an explicit exported C interface.

### 5.3 Root-level inputs and outputs

Root-level inputs and outputs should be included in the standard interface profile.

Their direction should be determined structurally from the model, not from the signal name. At the top level:

- a root-level `Inport` is an FMU input;
- a root-level `Outport` is an FMU output.

The preparation script should read the root port block type and port index, then write the direction into the generated registry. It should not add `_Input` or `_Output` to the model signal name merely to recover information that already exists in the model.

For nested subsystem ports, the block direction is only local to that subsystem. The FMU direction must be determined from the top-level interface boundary, not from every `Inport` or `Outport` block found recursively.

Keep these concepts separate in the registry:

```text
fmiCausality = input | output | parameter | local
xcpAccess    = read | write | readwrite
codeRole     = rootInput | rootOutput | parameter | internalSignal | state
```

For example, a root-level input is normally `fmiCausality=input` and may be writable through XCP. A root-level output is normally `fmiCausality=output` and is normally read-only. Both may be represented by generated C storage, but that does not make both FMU outputs.

The two wrapper records that currently represent one deactivated signal—one from the A2L and one from the DLL—should be joined by a stable `logicalId`. They should inherit the same registry metadata rather than independently defaulting to `output`.

For example:

```toml
[[variables]]
id = "root.in.vehicleSpeed"
active = true

[[variables]]
id = "root.out.vehicleSpeed"
active = false
```

The wrapper can retain separate `a2l` and `dll` backend records, but both records should point to the same `logicalId`, direction, type, dimensions, and access permissions.

If input and output display names are duplicated, use a collision-safe identity such as:

```text
root.in.vehicleSpeed
root.out.vehicleSpeed
```

or a generated identifier containing the model path and port index. The display name may remain unchanged. A suffix may still be used internally to guarantee unique A2L/C identifiers, but the wrapper must not infer semantic direction by parsing that suffix.

Record their:

- logical model path;
- direction;
- C identifier or generated structure field;
- data type;
- dimensions;
- sampling/update behavior;
- XCP read/write permissions;
- FMU causality.

Root-level data is preferable to arbitrary internal data because it already represents an explicit component interface.

### 5.4 Intermediate debug signals

Intermediate signals should be optional but preserved in a dedicated observable/debug build profile.

Do not expose every intermediate signal in the normal FMU. Instead, discover and preserve a controlled candidate set, for example:

- selected block outputs;
- outputs of selected subsystems;
- signals selected by a debug configuration file;
- signals needed by a known diagnostic or calibration workflow.

If the requirement truly is “all block outputs in debug mode,” preserve all addressable leaf outputs in the debug build, but still keep them disabled by default in the FMU wrapper.

## 6. Bus and virtual-signal policy

This is the most important rule for the current dimension error.

### 6.1 Virtual buses

A virtual bus is a grouping construct and does not necessarily have independent physical storage. Do not assign an exported storage class to the aggregate output of a virtual `Bus Creator` as if it were a scalar signal.

Instead, recursively enumerate the bus hierarchy and address the leaf signals:

```text
Bus1.r_N2_0_1
Bus1.r_N2_0_2
Bus1.r_N2_0_3
```

The preparation logic should behave conceptually as follows:

```text
if candidate is a leaf signal
    preserve and map the leaf
elseif candidate is a bus
    enumerate its children
    do not assign scalar StorageClass to the parent bus
else
    report an unsupported candidate
end
```

### 6.2 Nonvirtual buses

If the wrapper/XCP interface genuinely requires one structured bus object, define a matching `Simulink.Bus` object and make the bus nonvirtual deliberately.

Do not convert virtual buses to nonvirtual buses merely to make the generic script pass. That changes generated-code representation, memory layout, and possibly interface behavior.

### 6.3 Bus versus array

A Simulink bus is not the same thing as a numeric array:

- an array is accessed by index and has array dimensions;
- a bus is a named composite with named elements;
- a bus may contain vector or array leaves;
- an array of buses is a separate construct consisting of repeated nonvirtual buses with the same bus type.

Therefore, do not encode every bus as an array in the TQA, A2L, or FMU metadata. Represent it as a hierarchy:

```text
Bus1.signalA        scalar
Bus1.signalB        vector[4]
Bus1.Nested.signalC scalar
```

Only represent a signal as an array when its actual Simulink data type is an array/vector/matrix. Only represent an array of buses when the model explicitly creates an array of nonvirtual buses with a common `Simulink.Bus` type. [MathWorks signal types](https://www.mathworks.com/help/simulink/ug/signal-types.html) and [arrays of buses](https://www.mathworks.com/help/simulink/ug/combining-buses-into-an-array-of-buses.html) distinguish these cases.

For A2L/XCP, the most portable representation is usually one measurement or characteristic per addressable leaf. The bus parent does not need to be an `ExportedGlobal` merely because some of its leaves are measurable. If a nonvirtual bus is generated as a C structure, the registry can store the structure as a parent and store field offsets/addresses for its leaves.

The bus parent should be exported only when the wrapper or XCP implementation genuinely needs to read/write the complete structure as one object.

### 6.4 What the current error suggests

The reported error says that `Bus Creator1` is being forced to dimension `1` while `Bus Creator2` input 2 is also seen as a one-element vector. This is consistent with a preparation rule treating a bus aggregate as ordinary scalar output data.

The first preparation change should therefore be:

```text
exclude virtual/aggregate bus outputs from direct StorageClass assignment
traverse and preserve their addressable leaves instead
```

Do not use `CompiledPortDimensions` from a failed compilation as proof of the original signal dimensions.

## 7. Addressability and A2L generation

The DLL and A2L must be generated from the same final build.

The safest sequence is:

1. Prepare the temporary model.
2. Generate C code.
3. Compile and link the DLL.
4. Obtain final symbol/address information from the generated build.
5. Generate or customize the A2L using that final information.
6. Validate that each registry row corresponds to the final DLL/A2L.

Avoid calculating addresses from assumed structure offsets or manually computed padding whenever the code-generation toolchain can provide the information directly. Compiler alignment, structure packing, optimization, and build settings can change addresses.

Use the Simulink Coder ASAP2 workflow that matches the installed MATLAB release. In current releases, `coder.asap2.export` is the programmatic route for generating an ASAP2 file; older releases may use different configuration parameters or generation workflows. See [Configure Model Data Elements for ASAP2 File Generation](https://www.mathworks.com/help/rtw/ug/a2l-cal-profile.html) and [Generate ASAP2 and CDF Calibration Files](https://www.mathworks.com/help/rtw/ug/asap2-cdf-calibration.html).

For custom S-Function data that Simulink cannot describe automatically, add a small controlled post-processing layer. That layer should consume final symbols or explicit exported data, not guess addresses from model hierarchy alone.

## 8. Should you create a TQA/address table?

Yes, a table is a good idea if it is generated from the build and used as a registry. It should not be a second hand-maintained description that duplicates the A2L.

“TQA” does not appear to be a standard MathWorks or FMI term, so this document uses **data registry/address manifest** as the generic name. You can call it TQA in your tool if that is already the established terminology.

The registry should contain one row per addressable data item:

```text
logicalId
modelPath
busPath
name
direction / fmiCausality
codeRole
category
dataType
dimensions
readWriteAccess
storageClass
generatedCIdentifier
a2lIdentifier
ecuAddress
fmiValueReference
defaultActive
debugOnly
buildId
```

Recommended ownership:

- the registry owns the logical data inventory and wrapper metadata;
- the A2L owns ASAM/XCP-specific measurement and calibration descriptions;
- the DLL and linker/map output own the actual final addresses;
- the FMU wrapper owns which rows are active in a particular FMU.

The registry helps with:

- deterministic wrapper updates;
- activation/deactivation without regenerating Simulink code;
- validation of missing or duplicate addresses;
- mapping logical model paths to A2L names;
- preserving stable IDs while addresses change between builds;
- diagnosing unsupported data types and bus cases.

It adds very little generation time if it is emitted while the existing model-data/A2L information is already being collected. It becomes overcomplicated only if it independently reimplements A2L parsing, address calculation, and model-data discovery.

The registry should be generated, never manually edited.

### 8.1 S-Function parameter structures

Keeping S-Function parameters grouped in a generated C structure is reasonable. They do not need to become independent Simulink parameters merely to appear in the A2L.

Use a parent/field representation in the registry:

```text
parent:  SFunction1.Parameters
field:   SFunction1.Parameters.gain
type:    float64
offset:  0x20
address: parentAddress + offset
```

The field address must be derived from the final compiled structure layout. The current padding/offset mechanism can be retained, but it should be isolated as a layout resolver and validated against the actual compiler layout. The robust options are:

1. obtain the structure and field symbols from the final generated build;
2. generate a small C layout probe using the same compiler, headers, packing, and compiler flags, using `sizeof` and `offsetof`;
3. use Simulink/ASAP2 structure metadata when the installed release and A2L workflow can describe the structure directly.

Do not calculate offsets from MATLAB field order alone. Compiler alignment, packing pragmas, type sizes, and build options can change the result.

If the A2L tool cannot represent individual structure components, keep the parent structure in the A2L and keep the field-level layout in the registry used by the wrapper. If XCP must access each field independently, the A2L must ultimately contain enough component/type/offset information for the XCP implementation to address those fields.

## 9. FMU wrapper behavior

The wrapper should consume the registry rather than rediscovering model data.

For each requested FMU variable:

1. Look up its `logicalId`.
2. Verify that the build ID matches the DLL/A2L.
3. Verify the type and dimensions.
4. Assign or reuse an FMI `valueReference`.
5. Include it in `modelDescription.xml` only if selected.
6. Use the A2L/XCP mapping for runtime access.

The wrapper should fail clearly if a requested signal was not preserved:

```text
Requested signal is not present in this DLL build.
Rebuild with PreserveDebugSignals enabled.
```

It should not attempt to repair the model or reinterpret a bus as a scalar during packaging.

## 10. Build-time and runtime tradeoffs

Preserving additional data has real costs:

- more generated memory;
- more symbols and A2L entries;
- possible loss of signal-storage reuse;
- larger DLL and debug information;
- potentially longer compilation and linking;
- possible runtime overhead for copied or instrumented data.

The cleanest compromise is to provide two build profiles:

### Standard profile

- all required workspace/S-Function parameters;
- root-level inputs and outputs;
- no arbitrary intermediate signals;
- smaller and faster DLL.

### Observable/debug profile

- all standard-profile data;
- selected or all addressable intermediate leaf signals;
- complete registry/TQA;
- larger DLL and A2L;
- used when the wrapper must activate debug data without rebuilding the model.

If every future FMU must be able to activate every intermediate signal, the observable/debug profile must be used for every such DLL. There is no wrapper-only solution for a signal that was eliminated during code generation.

## 11. Safe preparation-script rules

The preparation script should:

1. Verify MATLAB and Simulink release.
2. Verify the required licenses and target files.
3. Verify the FMI Kit/plugin version.
4. Copy or load the model into an isolated build directory.
5. Save the original configuration snapshot.
6. Run a preflight model update before instrumentation.
7. Discover dependencies by category.
8. Reject unresolved workspace variables instead of silently guessing.
9. Preserve only supported leaf signals and data objects.
10. Avoid direct annotation of virtual/aggregate buses.
11. Use release-appropriate Code Mapping and ASAP2 APIs.
12. Generate code and link once.
13. Generate the A2L after final build information is available.
14. Generate the registry/TQA from the same data inventory.
15. Validate registry, DLL symbols, A2L entries, types, and dimensions.
16. Run a wrapper/XCP smoke test.
17. Close models and clean temporary files using guaranteed cleanup logic.

The script should fail closed. If it cannot determine whether a candidate is a leaf, bus, parameter, state, or unsupported custom object, it should report the item and stop rather than applying a generic storage class.

## 12. Functional validation

At minimum, validate the following:

### Model integrity

- the original model is unchanged;
- the prepared copy compiles before code generation;
- the prepared copy reaches code generation without new dimension errors;
- bus hierarchy and leaf counts are unchanged unless an explicit nonvirtual-bus conversion was requested.

### Generated data

- every required parameter has one registry row;
- every required root input/output has one registry row;
- every debug signal is a supported addressable leaf;
- every registry address resolves in the final DLL/map/A2L;
- type and dimension metadata agree across registry, generated code, and A2L;
- no two unrelated variables share an address unintentionally.

### Runtime interface

- XCP can read required measurements;
- XCP can write only permitted parameters;
- wrapper activation/deactivation changes only FMU metadata/interface selection;
- a debug FMU can read selected intermediate signals;
- a normal FMU does not expose debug-only data by default.

### Regression

Run the original and prepared model with the same test inputs and compare root-level outputs. Preservation and mapping should not change model behavior. Any bus conversion, inserted block, or signal-routing change requires separate behavioral review.

## 13. Remote licensed build and Python virtual environment

A Python virtual environment can be used for the orchestration layer, but it cannot replace the MATLAB/Simulink Coder license.

Simulink Coder code-generation configuration requires a Simulink Coder license. MATLAB can run non-interactively with `-batch`, and the license can be supplied through the normal license search path or a license-server option. See [MATLAB startup options](https://www.mathworks.com/help/matlab/matlab_env/startup-options.html) and [Build models from the command line](https://www.mathworks.com/help/rtw/ug/building-models-from-the-dos-window-command-line.html).

MATLAB Compiler is not a solution for packaging Simulink Coder generation; MathWorks lists Simulink Coder as unsupported by MATLAB Compiler and Simulink Compiler for this purpose. See [MathWorks Compiler support](https://www.mathworks.com/products/compiler/compiler_support.html).

### Recommended split

#### Local Python virtual environment

The local venv can provide:

- a CLI for build requests;
- configuration-file validation;
- packaging of the model/build inputs;
- submission to the internal remote-build tool;
- polling and log retrieval;
- artifact checksum validation;
- registry/TQA and A2L consistency checks;
- FMU packaging after the remote DLL/A2L build completes.

#### Remote licensed worker

The remote worker should provide:

- the required MATLAB release;
- Simulink and Simulink Coder;
- FMI Kit and required support packages;
- compiler/toolchain;
- custom libraries and S-Function dependencies;
- the required network or local license access;
- the MATLAB build entry point.

Conceptually:

```text
local venv
    -> submit build request
remote worker
    -> start matlab -batch
    -> prepare temporary model
    -> run slbuild/grt build
    -> generate DLL, A2L, registry, logs
    -> return an artifact package
local venv
    -> validate artifact package
    -> run wrapper and create FMU
```

The returned artifact package should include:

```text
DLL
A2L
data registry/TQA
generated-code/build identifier
model/configuration fingerprint
build log
tool/release/compiler information
```

The local venv should never assume that an address from an older build is valid for a new DLL. Addresses are build outputs; the registry and A2L must be regenerated together with the DLL.

## 14. Final recommendation

Implement the DLL build as a **category-aware maximal-addressable build**:

- preserve workspace parameters that are actually used;
- preserve supported S-Function parameters explicitly;
- always preserve root-level inputs and outputs;
- optionally preserve intermediate debug signals;
- recurse through buses and address leaves, not virtual aggregate outputs;
- generate the A2L from the final linked build;
- generate a registry/TQA from the same inventory;
- let the FMU wrapper activate only the selected registry rows;
- run MATLAB/Simulink Coder only on the licensed remote worker;
- use a local Python venv for orchestration, validation, and packaging.

This gives the wrapper maximum flexibility without requiring every model signal to appear in every FMU, avoids the current bus-dimension failure mode, avoids a clean-model double build in normal operation, and keeps address generation tied to the actual final DLL.

