# Project Context: SAP Code Extractor Toolset

## What this is

A custom SAP ECC (ABAP) toolset that extracts source-code changes from a transport request, program, class, or function module, formats them as a diff/change-log, and writes the result to a local text file. It's meant to give a reviewer (or an AI/external tool) a compact, dependency-annotated view of "what actually changed" without needing SAP GUI access to inspect version history object-by-object.

Repo (private): https://github.com/AjmalatWork/SAP-Code-Extractor
Local working copy: `D:\App Dev\SAP Code Extractor` (git repo, `main` branch)

## Components

### 1. `Report zcr_get_code_from_transport.txt` — ORIGINAL (as extracted from SAP)
The original driver report, written quickly/pragmatically, with logic inlined in `START-OF-SELECTION` rather than modularized. Kept in the repo as the historical baseline — do not edit this one.

### 2. `Report zcr_get_code_from_transport_REFACTORED.txt` — CURRENT, ACTIVE VERSION
A refactor of the above into the SAME report name (`REPORT zcr_get_code_from_transport.`) but restructured using modular `FORM`/`PERFORM` (not classes, by explicit design choice) so it's easier to extend. **This is the version confirmed to activate correctly in the SAP system and to produce byte-identical output to the original for the same transport request.**

Key structure:
- `START-OF-SELECTION` → `PERFORM main`.
- `main` orchestrates: determine extraction method/filename → build object list (dispatches to one of 4 source-specific FORMs) → add class methods → process each object → download file.
- Object-list builders: `build_object_list_from_tr`, `build_object_list_from_prog`, `build_object_list_from_class`, `build_object_list_from_func`.
- Per-object pipeline (`process_single_object`): `build_dependency_input` → `fetch_dependency_json` → `append_object_header` → `get_version_list` → `determine_comparison_mode` → `get_source_code` (new + old version) → `compute_delta` (or `build_full_delta` if only one version exists) → `append_differential_output` and/or `append_block_output`.
- Block-mode formatting splits further into `build_unified_code` (reconstructs a merged old+new line stream) and `extract_changed_blocks` (groups changes by enclosing `FORM`/`METHOD`/`FUNCTION`/`MODULE`/`DEFINE`, only emitting a block if something inside it changed).
- Error handling: instead of the original's raw `RETURN` calls scattered through inline code, uses a `gv_error` flag set by list-builder FORMs + `CHECK gv_error = abap_false` in `main`; individual per-object failures use `CHECK sy-subrc = 0` / `CHECK cv_skip = abap_false` inside FORMs, which — because a `CHECK` failing inside a FORM just exits that FORM — reproduces the original's `CONTINUE`-the-loop behavior naturally.

**Known quirk preserved intentionally (not a refactor bug):** in `build_dependency_input`, the `CPRI` (private section) branch calls `cl_oo_classname_service=>get_prosec_name` (protected-section getter) and `CPRO` (protected section) calls `get_prisec_name` (private-section getter) — these look swapped. This was already the behavior in the original report, so it was preserved rather than silently "fixed," pending confirmation from whoever owns the business logic.

**Fixed during activation:** `add_class_methods_to_object_list` exceeded ABAP's 30-character limit on FORM/subroutine names → renamed to `add_class_methods`.

### 3. `FUNCTION zcr_get_dependency_obj.txt` — dependency resolver (unchanged, external FM)
Given a list of `{obj_type, object_name}` pairs, resolves everything each object references:
- `PROG` → uses `REPOSITORY_ENVIRONMENT_RFC` to get the full environment (includes, function modules, DDIC objects, etc.)
- `FUGR` → function modules (`TFDIR`) + includes (`D010INC`) of the function group
- `CLAS` → methods (`SEOCOMPO`) + implemented interfaces (`SEOMETAREL`)
- `INTF` → interface methods
- `DOMA`/`DTEL`/`TABL`/`TTYP` → queued for DDIC metadata lookup
Serializes the dependency graph to JSON (`cl_sxml_string_writer` + `CALL TRANSFORMATION id`), and if any DDIC objects were touched, calls `Z_GET_DDIC_INFO` and merges its JSON into the output.

### 4. `FUNCTION z_get_ddic_info..txt` — DDIC metadata extractor (unchanged, external FM)
Given domain/data-element/table/table-type names, bulk-selects DDIC system tables (`DD01VV`, `DD02VV`, `DD03VV`, `DD04VVT`, `DD07V`, `DD12V`, `DD17S`, `DD22V`, `DD40VV`, `TADIR`) and builds, per development package: domain fixed values, data-element descriptions, table field/key/index/lock-object/SM30-maintainability info, structure layouts, and table-type line types. Serializes via `/ui2/cl_json=>serialize`.

### 5. `CODE_EXTRACT_DCSK9A7DFT_BLOCKS.TXT` — sample output
Real output from transport `DCSK9A7DFT`, generated in BLOCKS mode. Demonstrates the format: for each object, a header (`Object: <name> <type>`), the merged dependency+DDIC JSON, then only the changed `FORM` blocks (with `+`/`-`/`~`/`=` per-line markers). The sample transport itself adds a "SB Cxx notifications purge" feature (new checkbox `p_sb_cxx`, new branch in `FORM main_process` that submits a new purge report `zpcg1z_cms_purge_cxx_notif`), following the same pattern as existing purge-type checkboxes (CECR/trace/log/MS/history).

## How the report works (user-facing)

**Selection screen, block 1 — pick ONE source:**
- Transport Request (`p_tr`) — extracts every relevant object in the TR + its subtasks (filtered to `PROG, REPS, METH, CLAS, CLSD, CPUB, CPRI, CPRO, FUNC, DOMA, DTEL, TABL, TTYP`)
- Program (`p_prog`) — extracts the program + all its includes
- Class (`p_clas`) — extracts the whole class, or specific methods via `s_meth` if provided
- Function Module (`p_func`)

**Block 2:** the relevant input field per source above, plus `p_file` (output folder, browsable via F4).

**Block 3 — pick ONE output format:**
- **BLOCKS** (default) — diff grouped by changed FORM/METHOD/FUNCTION/MODULE, unchanged blocks omitted entirely. Most compact/useful for review.
- **DIFFERENTIAL** — flat line-by-line diff (`+`/`-`/`~`) between current and previous version.
- **FULL** — dumps the entire current source as if every line were new (no comparison against a previous version).

Output filename pattern: `CODE_EXTRACT_<TR|program|function|class>_<METHOD>.TXT`, written via `GUI_DOWNLOAD` to the chosen local folder.

## Environment constraints to keep in mind

- This is classic ABAP for an ECC system — no local compiler/interpreter exists; syntax check and activation only happen inside the real SAP system (SE38 → Syntax Check → Activate), or via ADT if connected to one.
- FORM/subroutine names: max 30 characters.
- Version comparison relies on SAP's built-in version-management function modules (`SVRS_GET_VERSION_DIRECTORY_46`, `SVRS_GET_REPS_FROM_OBJECT`, `SVRS_COMPUTE_DELTA_REPS`) — these only see what's in the version database, so newly-created objects with no prior version fall back to "first version" / FULL-like behavior.
- Output can contain client-identifiable details (custom object namespaces, business/process names, transport descriptions) — treat generated extracts and this repo as containing sensitive/proprietary content, not public-shareable code.

## Status as of now

- Refactored report activates cleanly in the real SAP system.
- Verified to produce the same extract output as the original for the same transport request (DCSK9A7DFT).
- Repo is private on GitHub, cloned to both a personal PC and a work PC; workflow is edit → commit → push (one machine) → pull (other machine).

## Where I want to take this (open for brainstorming)

*(Fill in specifics before pasting into the new chat — e.g.: richer dependency graphs, recursive extraction of dependent objects, HTML/Markdown output instead of flat text, integration with an LLM for auto-generated change summaries, support for more object types, a config/variant system instead of a single selection screen, unit-testable FORMs, etc.)*
