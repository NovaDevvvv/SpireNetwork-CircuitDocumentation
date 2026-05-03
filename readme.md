# Spire Network Chip Skript Guide

This document defines the minimum structure and recommended conventions for Spire Network chip skripts. The goal is to keep every chip readable, versioned, and predictable to maintain.

## Core Rules

Every chip should include all of the following:

1. A header comment that declares the chip name.
2. A complete port list in the header comment.
3. A `variables:` block.
4. A `_CHIP_VER` variable.
5. A `_CURRENT_EXEC` variable.
6. A trigger function named with the `CHIP_<CHIP_NAME>_TRIGGER` pattern.

If one of these pieces is missing, the chip should be treated as incomplete.

## Required File Layout

Use this order for every chip file:

```sk
# Chip: <Display Name>
# Ports:
#   Input 1 | Type: <type> | Name: <name>
#   Input 2 | Type: <type> | Name: <name>
#   Output 1 | Type: <type> | Name: <name>

variables:
    _CHIP_VER = <major>.<minor>.<patch>
    _CURRENT_EXEC = None

function CHIP_<INTERNAL_NAME>_TRIGGER(...):
    # chip logic
```

Keep the file structure stable. Other people should be able to scan any chip and immediately find its metadata, variables, and trigger logic in the same place.

## Header Contract

The top comment block is the chip contract. Keep it accurate.

### `# Chip:`

- Use the chip's display name.
- Keep the name aligned with the chip's actual behavior.
- Rename this if the chip's purpose changes.

### `# Ports:`

- List every port.
- Preserve the actual ordering used by the chip.
- Include whether the port is an `Input` or `Output`.
- Include the port index, type, and external name.
- Update the header any time a port is added, removed, renamed, or retyped.
- If a port can accept or emit multiple types, document it as `object`.

Recommended format:

```sk
# Ports:
#   Input 1 | Type: exec | Name: Run
#   Input 2 | Type: bool | Name: Condition
#   Output 1 | Type: exec | Name: Then
#   Output 2 | Type: exec | Name: Else
```

### Multi-Type Ports

If a port is allowed to hold more than one type, use `object` as the documented type.

Use `object` when:

1. The port can receive different value types depending on the graph setup.
2. The port can output different value types from the same chip.
3. The port is intentionally generic and should accept anything.

Do not list multiple runtime types in the header like `bool/float/string`. In the chip contract, those should be represented as `object`.

Example:

```sk
# Ports:
#   Input 1 | Type: object | Name: Value
#   Output 1 | Type: object | Name: Result
```

When a port is documented as `object`, the function parameter and return handling should also treat it as a generic value.

If the header says one thing and the function behaves differently, the header is wrong and must be fixed.

## Variables Contract

The `variables:` block must always exist, even for simple chips.

### `_CHIP_VER`

- Tracks the version of the chip implementation.
- Use `major.minor.patch` formatting.
- Increase the version whenever the chip is modified.

Use this versioning rule:

1. Increase `major` for breaking behavior or port contract changes.
2. Increase `minor` for new backwards-compatible behavior.
3. Increase `patch` for fixes, cleanup, or non-breaking internal adjustments.

Examples:

- `1.0.0` -> initial stable version
- `1.0.1` -> bug fix only
- `1.1.0` -> new compatible feature
- `2.0.0` -> breaking change to behavior or ports

### `_CURRENT_EXEC`

- Stores the currently triggered execution input.
- Prevents logic from running under the wrong execution path.
- Should be checked before branching into execution-specific behavior.

Treat `_CURRENT_EXEC` as part of the chip's execution guard, not optional state.

## Function Naming And Behavior

Use this pattern for the main trigger function:

```sk
function CHIP_<CHIP_NAME>_TRIGGER(...):
```

Guidelines:

- Keep the function name predictable and tied to the chip name.
- Match the function parameters to the data the chip actually consumes.
- Use `object` parameters for ports that may carry multiple value types.
- Guard execution-sensitive logic with `_CURRENT_EXEC` checks.
- Return the correct output identifier for the branch that should fire.

When returning a value:

- For execution-only outputs, return the output identifier such as `"output1"`.
- For outputs that carry data, return `"outputX|value"` using the engine's expected output format.

### Error Returns

Chips may also return engine error tokens when execution cannot continue normally.

Use this format:

```sk
"{error.<error_name>}"
```

Examples:

- `"{error.invalid}"`
- `"{error.internal_exception}"`

Use error returns when:

1. Input data is invalid for the chip's expected behavior.
2. A required value is missing or cannot be resolved.
3. An internal failure prevents the chip from producing a safe result.

Guidelines for error returns:

1. Use the engine error token directly, not an output label.
2. Keep the error name stable and descriptive.
3. Prefer a specific error such as `"{error.invalid}"` over a vague catch-all when the failure reason is known.
4. Use `"{error.internal_exception}"` for unexpected internal failures that the chip cannot recover from safely.
5. Document any chip-specific error behavior in comments near the relevant logic.

If a chip can fail in a known way, make that behavior intentional and readable instead of silently returning the wrong output.

Be consistent with output naming. If the engine expects `output1`, do not mix in another spelling later in the file.

## Robust Authoring Guidelines

Use these rules to make chips safer to maintain:

1. Keep names consistent between the header, variables, and function names.
2. Avoid undocumented ports or hidden behavior.
3. Keep branching logic explicit so the active path is obvious.
4. Update `_CHIP_VER` in the same change where behavior changes.
5. Prefer simple return paths over deeply nested logic when possible.
6. Do not leave stale comments after changing logic.
7. If a chip has fallback behavior, document it in comments near the logic.

## Example

```sk
# Chip: If
# Ports:
#   Input 1 | Type: exec | Name: Run
#   Input 2 | Type: bool | Name: Condition
#   Output 1 | Type: exec | Name: Then
#   Output 2 | Type: exec | Name: Else

variables:
    _CHIP_VER = 1.0.0
    _CURRENT_EXEC = None

function CHIP_IF_TRIGGER(input2 : bool):
    if _CURRENT_EXEC is "Input1":
        if input2:
            return "output1"
        else:
            return "output2"
```

What this example demonstrates:

1. The header fully documents the chip contract.
2. The required variables exist.
3. The trigger function follows the naming convention.
4. Execution is guarded by `_CURRENT_EXEC`.
5. The return values map directly to the declared outputs.

## Change Checklist

Before considering a chip update finished, verify all of the following:

1. The chip name in the header is still correct.
2. The port list matches the actual chip definition exactly.
3. The `variables:` block still exists.
4. `_CHIP_VER` was updated appropriately.
5. `_CURRENT_EXEC` is still used correctly for execution gating.
6. The trigger function name still matches the chip.
7. Every multi-type port is documented as `object`.
8. Every return path points to a valid output or a valid error token.
9. Any expected failure paths use the `{error.<name>}` format.
10. Comments still describe the current behavior.

## Common Mistakes

Avoid these issues:

1. Forgetting to update the top comment after changing ports.
2. Changing behavior without incrementing `_CHIP_VER`.
3. Returning the wrong output label for a branch.
4. Documenting a multi-type port as several concrete types instead of `object`.
5. Returning an ad-hoc error string instead of the `{error.<name>}` format.
6. Removing the `variables:` block because the chip looks simple.
7. Letting the chip name, function name, and behavior drift apart.

If you keep the contract, variables, versioning, and execution flow aligned, chips stay much easier to review and debug.

## Writing Documentation

Use [documentation.json](documentation.json) as the machine-readable reference for documented chips. This file should be updated whenever a chip is added, renamed, or has a behavior change that affects how people use it.

Each top-level property in [documentation.json](documentation.json) should be the chip's display name.

Recommended structure:

```json
{
    "Chip Name": {
        "usage": "# Chip Name\n\nShort Markdown documentation for the chip."
    }
}
```

### Required Documentation Field

Every documented chip should include this field in [documentation.json](documentation.json):

1. `usage`

### Usage Format

The `usage` value should be written as Markdown inside the JSON string.

Recommended Markdown sections inside `usage`:

1. A title heading with the chip name.
2. A short summary of what the chip does.
3. A `## Usage` section.
4. A `## Rules` section when behavior has important constraints.
5. A `## Errors` section when the chip can return error tokens.
6. A `## Example` section with a realistic sample.

### Documentation Rules

1. The chip name key must match the chip header name.
2. The `usage` value should be Markdown-formatted text.
3. Documentation should match the current ports and behavior exactly.
4. Multi-type values should be described as `object` in the explanation text.
5. If a chip can return error tokens, list them in the Markdown under an `Errors` section.
6. If a chip has execution-gating behavior, mention it in the Markdown rules or usage notes.
7. Keep wording user-facing. Document how to use the chip, not how the implementation happens internally unless that behavior affects usage.
8. Update [documentation.json](documentation.json) in the same change as the chip when behavior changes.

### Adding A New Chip

When adding a new chip to [documentation.json](documentation.json):

1. Add a new top-level object using the chip's display name.
2. Add a `usage` field containing Markdown text.
3. Document how to use the chip, any important rules, and any supported error tokens.
4. Add a realistic example that reflects the actual syntax.

### Documentation Review Checklist

Before finishing documentation changes, verify all of the following:

1. The chip exists as a top-level key in [documentation.json](documentation.json).
2. The chip name matches the actual chip name.
3. The `usage` field is valid Markdown text.
4. The usage text explains setup clearly enough for another person to follow.
5. Rules, errors, and example usage are included when they matter for the chip.
6. The content reflects current behavior.