
# GSoC 2026: Adding Prepared Statement Support to LuaSQL

*Google Summer of Code 2026 · LabLua*

---

## Introduction

I worked on adding prepared statement support to the
LuaSQL OCI8 driver as part of Google Summer of Code 2026.

## The API

The new API allows a statement to be prepared once
and executed with different parameters.

```lua
local stmt = conn:prepare(
    "INSERT INTO emp (id, name) VALUES (:id, :name)"
)

stmt:execute({
    id = {101, luasql.type.int},
    name = {"Risho", luasql.type.string},
})

stmt:close()
```

## Implementation Details

- Shared parameter validation
- Driver-agnostic type constants
- OCI8 parameter binding
- Prepared statement and cursor lifecycle management

## Testing

Describe the tests, edge cases, and debugging work
you completed during the project.

## What's Next?

The next step is to extend the same API to other
LuaSQL drivers.

## Links

- [Tracking issue](https://github.com/lunarmodules/luasql/issues/207)
- [Pull request](https://github.com/lunarmodules/luasql/pull/209)

## Acknowledgements

Thanks to Tomás Guisasola, Thijs Schreijer,
and the LuaSQL community.
