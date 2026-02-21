# Roadmap

Ideas that are too good (or too cursed) to forget but not urgent enough to implement now.

For the emulation boundary — what can cross the bridge and what can't — see [Concepts](developer/concepts.md#the-emulation-boundary).


## Next

### The distribution becomes a manifest

Every extension currently bundled in the helmfile2compose distribution (`workloads.py`, `caddy.py`, `haproxy.py`, indexers) becomes its own repo, its own extension referenced in `extensions.json`. The `helmfile2compose` repo shrinks to a single `distribution.yaml` — a manifest that `build-distribution.py` reads, calls h2c-manager with a new `--fetch-extensions-only` flag to download everything, and assembles the single-file script.

`helmfile2compose` stops being code and becomes a shopping list. It will forever carry the short but horrifying history of a project that started from a simple need and became far, *far* too complicated for what it was supposed to be. The release? Not a 4.0. Not even a 3.x.0. A 3.x.y+1 patch release — because the output won't change by a single comma. Just the location of said code.

### The distribution family

Three distributions that stack on top of each other.

```
h2c-core (bare engine, ~1265 lines)
  └─ helmfile2basic (+ indexers, workloads)
       └─ helmfile2compose (+ caddy, haproxy)
            └─ helmfile2easy (+ every official extension)
```

| Distribution | What's in it | What it is |
|---|---|---|
| **helmfile2basic** | h2c-core + indexers + WorkloadConverter | A shiv. Services, ConfigMaps, Secrets, PVCs. No reverse proxy, no opinions. BYOE. |
| **helmfile2compose** | helmfile2basic + Caddy + HAProxy rewriter | A revolver. The OG. Ingress → Caddy, h2c-manager for the rest. |
| **helmfile2easy** | helmfile2compose + all official extensions | A rocket launcher. One script, one page of docs, zero configuration. |

Requires: stacking support in `build-distribution.py` (build on top of a parent distribution, not just bare h2c-core), extracting the 7 built-in extensions into their own repos.

### Gateway API

The Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) is the eventual successor to Ingress — `HTTPRoute`, `Gateway`, `GRPCRoute` instead of `Ingress`. A `GatewayRewriter` extension would handle these kinds the same way `IngressRewriter` handles Ingress annotations: read the Gateway API resources, produce Caddy config. No rush — Gateway API adoption is still ramping up — but the extension system should be ready for it when it comes.

### Pipeline hooks

Named pipeline hooks (`post_aliases`, `pre_write`, etc.) for extensions. Not needed yet — converters + transforms cover known cases. Revisit if a third pattern shows up.

### Rename `caddy_entries` to `ingress_entries`

`ConvertResult.caddy_entries`, `disableCaddy`, `caddy_email`, `caddy_tls_internal` — these names leak the default provider into the core API and config schema. Now that `IngressProvider` is an abstract base class and the system is provider-agnostic, rename to generic terms (`ingress_entries`, `disable_ingress`, `ingress_email`, `ingress_tls_internal` or similar). Breaking change for extensions that reference `caddy_entries` directly — coordinate with a major version bump or a deprecation period.

### Config key naming consistency

`helmfile2compose.yaml` mixes camelCase (`disableCaddy`, `ingressTypes`, `helmfile2ComposeVersion`) and snake_case (`volume_root`, `caddy_email`, `caddy_tls_internal`, `core_version`). Normalize to one convention (snake_case, with camelCase aliases for backwards compatibility during a transition period). Overlaps with the `caddy_entries` rename above — do both at once.

### Extension compatibility matrix

Extension manifests with `core_version_min` / `core_version_max_tested`. Manager warns/errors on mismatch. Today only extension-vs-extension incompatibility is checked — extension-vs-core version compat is not.

### More extensions

New CRD extensions (providers and converters) and transforms as needed. The extension system exists — writing a new one is straightforward (see [Writing converters](developer/extensions/writing-converters.md) and [Writing transforms](developer/extensions/writing-transforms.md)).

## v3.0 — contract split ✓ {#v30--contract-split}

**Shipped in v3.0.0.** The provider/converter distinction is now enforced via base classes: `Converter`, `Provider` (produces services), `IndexerConverter` (populates ctx), and `IngressProvider` (reverse proxy backends). The package was renamed from `helmfile2compose` to `h2c` internally, and `build-distribution.py` was introduced for building distributions. `_auto_register()` scans globals and registers all extension classes, with fatal duplicate-kind detection. See the [journal](journal.md#v300-the-separation-of-worlds) for the full story.

## Out of scope

CronJobs, resource limits, HPA, PDB, probes-to-healthcheck. See [Limitations](limitations.md) for the full list and rationale.

---

> *Thus spoke the disciple unto the void: "Yog Sa'rath, my hour has come." And the void answered not — for even it knew that some invocations are answered not with knowledge, but with consequences.*
>
> — *De Vermis Mysteriis, On the Hubris of the Disciple (probably³)*

*For what's already been done, see the [cursed journal](journal.md).*