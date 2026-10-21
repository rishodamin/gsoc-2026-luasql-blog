# Adding Prepared Statement Support to LuaSQL: GSoC 2026

*Google Summer of Code 2026 · LabLua · Contributor: Damin Risho J V*

---

## Introduction

During Google Summer of Code 2026, I worked with LabLua on adding prepared statement support to LuaSQL, starting with the Oracle OCI8 driver.

LuaSQL provides a simple interface for connecting Lua applications to different database systems. My project focused on introducing a common prepared statement API, along with a shared datatype system and parameter validation, while keeping the implementation compatible with the existing LuaSQL design.

The OCI8 implementation is complete, including parameter binding, statement execution, cursor integration, tests, and documentation. The [pull request](https://github.com/lunarmodules/luasql/pull/209) is ready for review.

**Project links**

- [Tracking issue and API design](https://github.com/lunarmodules/luasql/issues/207)
- [Pull request #209](https://github.com/lunarmodules/luasql/pull/209)
- [LuaSQL repository](https://github.com/lunarmodules/luasql)


## Project Demo

Here's a short walkthrough of my GSoC 2026 project, demonstrating
prepared statement execution in LuaSQL's OCI8 driver.

<p align="center">
  <a href="https://youtu.be/phnK7mW26UM">
    <img
      src="https://img.youtube.com/vi/phnK7mW26UM/maxresdefault.jpg"
      alt="Watch the LuaSQL prepared statements demo on YouTube"
      width="800"
    />
  </a>
</p>

<p align="center">
  <a href="https://youtu.be/phnK7mW26UM">
    <strong>▶ Watch the demo on YouTube</strong>
  </a>
</p>

---


## The Problem

LuaSQL already provides an interface for executing SQL statements and retrieving query results. The goal of this project was to add prepared statements with parameter binding through a consistent API.

A common API is useful because application code should not have to learn an entirely different interface for every database driver. At the same time, each driver still needs to map the common LuaSQL types and operations to the facilities provided by its underlying database library.

The first target was OCI8, which uses Oracle's OCI API. This involved more than preparing SQL: the implementation also had to handle parameter conversion, statement and cursor lifecycles, error handling, and resource cleanup safely.

## The API

The API was designed around three operations:

- `conn:prepare(sql)` prepares a statement.
- `stmt:execute(params)` binds the supplied parameters and executes the statement.
- `stmt:close()` releases the prepared statement's resources.

Here is an example using named parameters:

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

Parameters carry an explicit LuaSQL datatype. This allows the driver to convert each value to the appropriate database representation.

The API also supports positional parameters where the driver supports them:

```lua
stmt:execute({
    [1] = {101, luasql.type.int},
    [2] = {"Risho", luasql.type.string},
})
```

Named and positional parameters are alternative ways to supply parameters; they should not be mixed in the same parameter table.

For `SELECT` statements, `stmt:execute()` returns a cursor that works with the existing LuaSQL cursor methods, such as `fetch()` and `getcolnames()`. For DML statements, execution returns the driver's affected-row result.

## What I Implemented

### 1. A common datatype system

Instead of exposing database-specific constants directly, the API defines driver-independent types through `luasql.type`.

Examples include:

- `luasql.type.int`
- `luasql.type.number`
- `luasql.type.string`
- `luasql.type.boolean`
- `luasql.type.date`
- `luasql.type.time`
- `luasql.type.timestamp`
- `luasql.type.null`

Each driver can map these common types to its own internal database types. This keeps the Lua-facing interface consistent while allowing the underlying implementations to differ.

### 2. Shared parameter validation

I added a shared `luasql_validate_params()` helper in `luasql.c` to validate the parameter table before binding begins.

Keeping validation in a common layer avoids duplicating the same parameter checks in every driver and provides a foundation for extending prepared statement support to other databases.

### 3. OCI8 parameter binding and execution

I implemented parameter binding and execution for the OCI8 driver, including named and positional parameters and support for typed values, NULL, dates, and timestamps.

The implementation handles both DML statements and queries that return result sets. For queries, it integrates with LuaSQL's existing cursor interface rather than introducing a separate way to fetch results.

### 4. Statement and cursor lifecycle management

Prepared statements introduce additional resource-management concerns. A statement may outlive an individual execution, and a cursor may depend on a statement that was prepared earlier.

I added statement infrastructure to manage the OCI statement handle, error handle, parent connection reference, and cursor state. Cursor ownership is tracked so that a cursor using a prepared statement does not incorrectly free a handle owned by that statement.

The implementation also handles cleanup when statements are closed and prevents a prepared statement from being closed while its cursor is active.

## Interesting Implementation Details

### Binding and execution in one call

An early API design used separate `stmt:bind()` and `stmt:execute()` calls. During the design discussions, I changed this to accept parameters directly in `stmt:execute(params)`.

This avoids maintaining a separate persistent set of bound parameter buffers between calls and makes the API easier to use.

### Handling Oracle dates

Oracle date conversion can depend on session settings such as `NLS_DATE_FORMAT` when string-based conversions are used.

For date parameters, I convert ISO-formatted date strings into native `OCIDate` structures and bind them using `SQLT_ODT`. This avoids depending on the session's date format for this conversion.

### Avoiding double-free errors

A prepared statement and a cursor can refer to the same underlying OCI statement handle, but they do not both own that handle.

The implementation tracks the relationship between the cursor and its prepared statement using a Lua registry reference. This lets the cleanup logic distinguish a borrowed statement handle from one owned by the cursor, preventing the same resource from being released twice.

### Memory safety and cleanup

Code review also led to improvements in allocation order, error handling, and cleanup after partial failures. I fixed a counter-underflow issue and improved cleanup logic so that resource management remains consistent when an operation fails.

These changes were important because database drivers must handle not only successful execution but also failures at different stages of statement preparation and execution.

## Testing and Debugging

Testing was an important part of the implementation. I added prepared statement tests and datatype constant tests, and fixed existing OCI8 test issues encountered during development.

The tests cover areas including:

- DML and `SELECT` execution.
- Named and positional parameters.
- Parameter validation and invalid inputs.
- Typed parameter binding, including NULL, dates, and timestamps.
- Cursor cleanup and statement lifecycle handling.
- Datatype constants and error cases.

I also investigated issues involving automatic cursor cleanup, transaction rollback behavior, and OCI8-specific test expectations.

One important lesson was that database drivers can have behavior that differs from what a generic interface might suggest. Tests need to account for those differences without making the common API unnecessarily complicated.

## What's Next?

The OCI8 implementation establishes the initial prepared statement API, datatype system, shared validation layer, and tests.

A natural next step is to extend the same interface to other LuaSQL drivers:

- PostgreSQL
- SQLite3
- MySQL

The shared infrastructure should make those implementations easier to develop, although each driver will still need its own parameter-binding logic, datatype mappings, and tests.

The current implementation and API design are documented in [issue #207](https://github.com/lunarmodules/luasql/issues/207), and the OCI8 changes are available in [pull request #209](https://github.com/lunarmodules/luasql/pull/209).

## Acknowledgements

Working with LabLua was a great learning experience. The project gave me the opportunity to work on C code, Oracle's OCI API, database driver internals, resource ownership, and API design in an established open-source project.

A special thanks to **Tomás Guisasola** for his patient mentorship, **Thijs Schreijer (Tieske)** and **Lourival Vieira Neto** for their guidance, detailed code reviews, and feedback throughout this project. Their guidance helped me think more carefully about design decisions, error handling, and maintainability.

I'm grateful to the LabLua and LuaSQL community for the opportunity to contribute and learn through Google Summer of Code 2026.

---

*GSoC 2026 · LabLua · LuaSQL*
