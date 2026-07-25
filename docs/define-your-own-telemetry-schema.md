# Define your own telemetry schema

Weaver allows you to specify your application's signals using a telemetry schema
based on the concept of semantic conventions. These telemetry schemas are also
called custom registries. You can define your own signals or reuse the signals
and attributes defined by the OTEL semantic conventions (url).

To define your schema, you must:

- create a directory that will contain your semantic conventions
- add a `manifest.yaml` file (see structure below)
- add semantic convention files describing the signals that your application may
  produce

From there, you can use Weaver commands to check, resolve, generate code and
documentation, or even control your instrumentation with the `live-check`
command.

## `manifest.yaml` file

This manifest file is used to define the metadata of your custom registry and
the dependencies it has on other registries. The manifest file is required for
the `weaver` tool to recognize your custom registry and to resolve it correctly.

```yaml
name: <custom registry name>
description: <an optional description of the custom registry>
schema_url: <base URL where the registry's schema files are hosted>/<version of this custom registry>
dependencies:
  - name: <an alias for the dependency>
    registry_path: <the location of the dependency>
```

> **Current limitations**:
> - Weaver supports a maximum of 10 registry levels without circular
    dependencies. In practice, this is not a limitation, even for complex
    enterprise environments.

Below is an example of a valid `manifest.yaml` file:

```yaml
name: acme
description: This registry contains the semantic conventions for the Acme vendor.
schema_url: https://acme.com/schemas/0.1.0
dependencies:
  - name: otel
    registry_path: https://github.com/open-telemetry/semantic-conventions@v1.40.0[model]
```

## Semantic conventions files

Semantic conventions are defined in YAML files. OpenTelemetry maintains a
registry of general attributes and signals across many domains, from databases
and messaging to generative AI (see
the [official OTEL semantic conventions](https://opentelemetry.io/docs/specs/semconv/)).

You can define your own semantic conventions or reuse those defined by OTEL. The
example below shows how to define a simple span semantic convention representing
a message sent by a client application.

You can also import existing semantic conventions from other registries, such as
the OTEL semantic conventions, to extend your custom registry.

```yaml
groups:
  - id: span.example_message
    type: span
    stability: development
    brief: This span represents a simple message.
    span_kind: client
    attributes:
      - ref: host.name                 # imported from OTEL semantic conventions
        requirement_level: required    # requirement level redefined locally
      - ref: host.arch                 # imported from OTEL semantic conventions
        requirement_level: required    # requirement level redefined locally

imports:
  metrics:
    - db.*                # import all metrics in the `db` namespace
  entities:
    - gcp.*               # import all entities in the `gcp` namespace
  events:
    - session.start       # import the `session.start` event
```

In the `imports` section, you can specify which metric, event, and entity groups
to import from other registries. You can use a wildcard expression to import all
groups in a namespace or specify individual groups by name.

## Run weaver commands on your schema

To check the validity of your custom registry

```bash
weaver registry check -r <path-to-your-registry>
```

To control the compliance of your instrumentation with your custom registry

```bash
weaver registry live-check --registry <path-to-your-registry>
```

All commands accepting the `-r` or `--registry` parameter can be applied to your
custom registry. It is important to note that some templates are specific to the
OTEL registry. We are working to remove this type of limitation.

The `<path-to-your-registry>` parameter can be a local directory, a local or
remote archive, a remote file URL (such as a published registry manifest), or a
Git URL. GitHub release asset URLs are also supported and are automatically
resolved via the GitHub API.

It is also possible to use specific Git references, such as a tag, a branch or
even a specific commit with the `<path-to-your-registry>@<refspec>` syntax.

To download from private sources, configure HTTP authentication per-URL in
`.weaver.toml` with one or more `[[auth]]` entries. Each entry pairs a
`url_prefix` (longest match wins) with a token source — a literal `token`,
a `token_env` variable name, or a `token_command` helper whose first stdout
line is the token:

```toml
[[auth]]
url_prefix    = "https://github.com/"
token_command = ["gh", "auth", "token"]
```

Matching requests are sent with `Authorization: Bearer <token>`. Then:

```bash
weaver registry check \
  -r "https://github.com/org/repo/releases/download/v1.0.0/manifest.yaml"
```

## Caching pinned Git registries

By default, every `weaver` invocation re-clones a remote Git registry (and every
remote `dependencies[]` entry of a manifest) into a throwaway temporary
directory that is deleted when the command exits. When many invocations resolve
the same registry — for example, one `weaver` container per integration-test
suite — this repeated cloning dominates start-up time.

To avoid it, weaver can cache a **version-pinned** Git registry in a shared
directory and reuse it across invocations. A source is considered pinned when it
carries an explicit `@<refspec>` (a tag, branch, or commit); a bare URL that
tracks a moving default branch is never cached, so it can never serve stale
content. The cache is controlled by these global CLI flags (available on every
subcommand):

| Flag | Effect |
| --- | --- |
| `--registry-cache-dir <PATH>` | Cache root directory. Providing it enables the cache; omitting it keeps the default throwaway-clone behavior. |
| `--offline` | A cache miss for a pinned source is a hard error instead of a network fetch — for deterministic CI runs. |
| `--registry-cache-refresh` | Re-fetch and atomically replace a cached entry even on a hit — for the rare case of a moving tag or branch. |

Cache entries are keyed by `(url, refspec)`, so registries that differ only by
sub-folder share a single clone. Population is concurrency-safe: the clone is
staged in a private directory and then atomically moved into place, so parallel
`weaver` processes never observe a half-populated entry and a lost race simply
reuses the winner's clone. `--offline` takes precedence over
`--registry-cache-refresh` (offline never re-fetches). Prefer running a refresh
while no other `weaver` process is reading the same cache entry.

```bash
# Populate (or reuse) the cache, then run fully offline against the pinned copy.
weaver --registry-cache-dir /path/to/weaver-cache \
  registry check -r "https://github.com/open-telemetry/semantic-conventions.git@v1.41.0[model]"
weaver --registry-cache-dir /path/to/weaver-cache --offline \
  registry live-check --registry "https://github.com/open-telemetry/semantic-conventions.git@v1.41.0[model]"
```

In CI, point `--registry-cache-dir` at a cached/restored directory keyed on the
pinned version, and add `--offline` so a cache miss fails loudly rather than
silently reaching the network.
