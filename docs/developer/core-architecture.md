# Core architecture

> *The disciple gazed upon the monolith and saw that it was vast — a single tablet bearing every law, every rite, every contradiction. And he said: let us shatter the tablet, that each fragment may be understood alone. But when the pieces lay scattered, each fragment still whispered the name of the whole.*
>
> — *Necronomicon, On the Shattering of Tablets (fragmentary)*

## Why split

`helmfile2compose.py` grew to ~1860 lines — a single file containing types, constants, parsers, converters, CLI, and output writers. It worked, but navigating it required a mental index. The split preserves the single-file distribution (users and h2c-manager see no change) while making the codebase navigable.

## Package structure

```
h2c-core/
├── src/h2c/
│   ├── __init__.py          # re-exports pacts API (backwards compat)
│   ├── __main__.py          # python -m h2c entry point
│   ├── cli.py               # argument parsing, orchestration
│   ├── pacts/               # public contracts for extensions
│   │   ├── __init__.py
│   │   ├── types.py         # ConvertContext, ConvertResult, Converter, Provider
│   │   ├── ingress.py       # IngressRewriter, get_ingress_class(), resolve_backend()
│   │   └── helpers.py       # apply_replacements(), _secret_value()
│   ├── core/                # internal conversion engine
│   │   ├── __init__.py
│   │   ├── constants.py     # regexes, kind lists, well-known ports
│   │   ├── env.py           # resolve_env(), port remapping, command conversion
│   │   ├── volumes.py       # volume mount conversion (PVC, ConfigMap, Secret)
│   │   ├── services.py      # K8s Service indexing, alias maps, port maps
│   │   ├── ingress.py       # IngressProvider, rewriter dispatch, _NullRewriter
│   │   ├── extensions.py    # extension discovery and loading
│   │   └── convert.py       # convert() orchestration, overrides, fix-permissions, _auto_register()
│   └── io/                  # input/output
│       ├── __init__.py
│       ├── parsing.py       # helmfile template, YAML loading, namespace inference
│       ├── config.py        # helmfile2compose.yaml load/save
│       └── output.py        # compose.yml, Caddyfile, warnings
├── build.py                 # concat → single-file h2c.py (bare engine)
├── build-distribution.py    # concat core + extensions → distribution (e.g. helmfile2compose.py)
└── .github/workflows/
    └── release.yml          # CI: build + release assets on tag push
```

Note: `workloads.py` (WorkloadConverter) and `haproxy.py` (HAProxyRewriter) are **not** in h2c-core — they are distribution extensions, bundled in the [helmfile2compose](https://github.com/helmfile2compose/helmfile2compose) distribution's `extensions/` directory. The bare core has empty registries; the distribution populates them via `_auto_register()`.

## The three layers

### `pacts/` — the sacred contracts {#pacts--the-sacred-contracts}

Everything extensions can import. This is the stable API:

```python
from h2c import ConvertContext, ConvertResult
from h2c import IngressRewriter, get_ingress_class, resolve_backend
from h2c import apply_replacements, resolve_env
```

Or explicitly:

```python
from h2c.pacts import ConvertContext, IngressRewriter
```

Both paths work — `__init__.py` re-exports the pacts API. These are the only imports extensions should use. If it's not in `pacts/`, it's internal and may change.

### `core/` — the conversion engine

Internal modules that do the actual work. Not part of the public API. The dependency graph is acyclic:

```
constants.py      ← standalone
env.py            ← pacts/helpers, constants
volumes.py        ← pacts/helpers, env
services.py       ← constants
ingress.py        ← pacts, IngressProvider base class
extensions.py     ← pacts/ingress, ingress
convert.py        ← pacts, all core modules, _auto_register()
```

Module-level mutable state lives in `core/convert.py` (`_CONVERTERS`, `_TRANSFORMS`, `CONVERTED_KINDS`) and `core/ingress.py` (`_REWRITERS`). In the distribution model, `_auto_register()` populates these registries from all classes found in the concatenated script's globals.

### `io/` — input/output

Parsing, config, and output writing. No conversion logic — just plumbing between the filesystem and the core.

## Build / concat

Two build scripts, two purposes:

- **`build.py`** — concatenates `src/h2c/` into a single `h2c.py`. Bare engine, no extensions. This is the h2c-core release artifact.
- **`build-distribution.py`** — builds a distribution from core + extensions dir. Used by distribution repos (e.g. [helmfile2compose](https://github.com/helmfile2compose/helmfile2compose)).

Both follow the same pattern:

1. Reads modules in manually-maintained dependency order (constants first, CLI last) — the `MODULES` / `CORE_MODULES` list must respect the import graph. A topological sort would automate this, but 15 lines that change once a year don't warrant a DAG resolver. If the order is wrong, the `--help` smoke test fails immediately.
2. Strips internal `from h2c...` imports (everything's in one namespace)
3. Deduplicates stdlib/external imports
4. Writes the output file with shebang + docstring + all code
5. Runs `--help` as a smoke test

```bash
# Bare core
python build.py
# → h2c.py (build artifact, not committed)

# Distribution
python build-distribution.py helmfile2compose --extensions-dir extensions --core-dir ../h2c-core
# → helmfile2compose.py
```

The CI workflow (`.github/workflows/release.yml`) runs `build.py` on tag push and uploads `h2c.py` + `build-distribution.py` as release assets.

See [Build system](build-system.md) for the full deep dive.

## Dev workflow

Assumes `h2c-core` and `h2c-testsuite` are checked out as sibling directories.

```bash
# Run from the package (development)
cd h2c-core
PYTHONPATH=src python -m h2c --from-dir /tmp/rendered --output-dir /tmp/out

# Build the bare core (testing core artifact)
python build.py
python h2c.py --from-dir /tmp/rendered --output-dir /tmp/out

# Validate with the executioner
cd ../h2c-testsuite
./run-tests.sh --local-core ../h2c-core/h2c.py
```

Both paths produce identical output — the executioner validates this.
