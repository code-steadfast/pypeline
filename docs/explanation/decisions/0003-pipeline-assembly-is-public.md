# ADR 0003: Include expansion is public API for embedding applications

**Status:** Accepted (implemented, pending release)

## Context

Pypeline supports being used as a library. `PipelineConfig` and `PipelineLoader[T]`
are generic, and {doc}`../../how_to/use_as_library` tells a consumer that the config
file format is up to them, because "`PipelineConfig` just needs a dict".

An application that follows that advice parses its own file and loses everything the
loader adds on top of the raw dict:

- `include:` entries are never expanded, so they reach the scheduler with no `step`
  and no `class_name` to resolve;
- output groups are never stamped, so an included step's cache identity is wrong;
- entry validation and circular-include detection never run.

All of it lived inside `ProjectConfig._load`, and `_load` parses the file as a
`ProjectConfig`, whose `pipeline` field is required. An application with its own
schema had no way in. The result was a quiet failure: the config is valid, the run
starts, and the included steps are simply absent.

## Decision

One public function:

```python
def assemble_pipeline(pipeline: PipelineConfig, source_file: Path) -> PipelineConfig
```

It stamps output groups, validates entries and expands includes for a pipeline that
was parsed from `source_file` by any schema. `ProjectConfig.from_file` is its first
caller, which is what keeps the embedded path and pypeline's own path from drifting.

The expansion helpers moved from `ProjectConfig` classmethods to module-level
functions. `ProjectConfig._load` stays as the fragment loader, so cycle detection
lives in one place, on the path every fragment takes.

**Include paths stay relative to the including file.** No resolver hook is provided.
An absolute `include:` path is used as given, so an application with its own file
lookup resolves the path itself and passes an absolute one.

## Alternatives considered

- **A resolver callback threaded through the expansion.** Rejected for now. The only
  case it serves is a *fragment* including another fragment by a path relative to
  neither itself nor its includer. Absolute paths cover the rest, and a hook is easy
  to add once a second consumer needs one.
- **A dedicated dataclass for fragments** instead of loading them as `ProjectConfig`.
  Rejected for now: `ProjectConfig` requires only `pipeline`, its one extra field is
  optional, and unknown keys are ignored, so a separate type would change no
  behaviour. Revisit when `ProjectConfig` gains a required field a fragment should
  not have to satisfy.

## Consequences

- `assemble_pipeline` is API. It is covered by tests that never construct a
  `ProjectConfig`, so a refactor of the loader cannot silently break embedders.
- A fragment is a file with a top-level `pipeline:` key. Its steps may be typed to the
  embedding application's own step base class, in which case the fragment is loadable
  by that application only, not by `pypeline run`. That limit is documented in the
  how-to rather than enforced.
- A consumer that ignores `assemble_pipeline` keeps today's behaviour, which is why
  this is additive.
