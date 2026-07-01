# Architecture

app-common-python is a schema-driven Python client library that provides access to [Clowder operator][clowder-project] configuration in Red Hat Insights applications. The library's design centers on automatic code generation from a canonical JSON schema, ensuring type safety and schema compatibility without manual synchronization.

## Design Overview

The library follows a **schema-as-source-of-truth** model. All Python types are generated from the upstream Clowder JSON schema, with no hand-written data models. This approach ensures that configuration structure changes in the Clowder project automatically propagate to client code through regeneration.

### Key Components

1. **Schema Synchronization** — The [sync_config.sh script][sync-script] fetches the canonical schema from the Clowder project's master branch and triggers code generation.
2. **Code Generation** — The [yacg templating engine][yacg-project] transforms the JSON schema into Python dataclasses via the `pythonBeans` template.
3. **Runtime API** — The [LoadedConfig module][loadedconfig-module] provides convenience accessors and lazy initialization for certificate data.
4. **Packaging** — Modern [pyproject.toml-based packaging][pyproject-file] with git-tag-based versioning via setuptools_scm.

## Code Generation Pipeline

### Schema Dependency

The library maintains a vendored copy of the Clowder schema at [schema.json][schema-file]. This file is the **sole source** for all type definitions. The schema defines the complete `AppConfig` structure, including nested objects for Kafka brokers, database credentials, object storage buckets, and service endpoints.

The schema is fetched from:
```
https://raw.githubusercontent.com/RedHatInsights/clowder/master/controllers/cloud.redhat.com/config/schema.json
```

### Generation Workflow

The [sync_config.sh][sync-script] script executes two steps:

1. **Download** — Fetches the latest schema from Clowder's master branch and overwrites the local `schema.json`.
2. **Generate** — Runs the yacg Docker image (`okieoth/yacg:latest`) with:
   - Input model: `/resources/schema.json`
   - Template: `pythonBeans` (built into yacg)
   - Output: `/resources/app_common_python/types.py`

The script uses Podman with a volume mount (`-v pwd:/resources:Z`) to provide the local repository as the working directory. The `:Z` flag handles SELinux labeling.

### Generated Code Structure

The [types.py file][types-file] contains:

- A warning header: `"Attention, this file is generated. Manual changes get lost with the next run of the code generation."`
- One Python class per schema definition (e.g., `AppConfig`, `KafkaConfig`, `BrokerConfig`, `DatabaseConfig`)
- `dictToObject` class methods for JSON deserialization
- Initialization of all fields to `None` or empty lists

All classes are plain Python objects with no external dependencies. The generated code includes docstrings extracted from the schema's `description` fields.

### Why yacg?

The library uses [yacg (Yet Another Code Generator)][yacg-project] because:

1. **Template-based** — The `pythonBeans` template generates idiomatic Python dataclasses with `dictToObject` deserializers, matching the library's API design.
2. **JSON Schema native** — yacg reads JSON Schema Draft 7 directly without conversion.
3. **Containerized** — The Docker image pins the template version and eliminates local Python dependencies for the generation step.
4. **Single-file output** — The `--singleFileTemplates` flag consolidates all types into one module, avoiding import fragmentation.

Alternative approaches (e.g., dataclasses-json, pydantic) would require runtime dependencies and manual schema translation.

## Runtime Architecture

### Configuration Loading

The [LoadedConfig module][loadedconfig-module] provides the primary API. On import, it:

1. Reads the `ACG_CONFIG` environment variable (expected to point to a JSON file path).
2. Loads the JSON file (or an empty dict if the variable is unset).
3. Deserializes the JSON into a `SmartAppConfig` instance via `dictToObject`.

`SmartAppConfig` extends the generated `AppConfig` class with two helper methods:

- **`rds_ca()`** — Writes the RDS CA certificate (from `self.database.rdsCa`) to a temporary file and returns the path. Caches the path in `self._rds_ca` to avoid redundant writes.
- **`kafka_ca()`** — Writes the Kafka CA certificate (from `self.kafka.brokers[0].cacert`) to a temporary file and returns the path. Caches the path in `self._kafka_ca`.

These methods address a common pattern: many Python database and Kafka libraries expect CA certificates as file paths, not inline strings.

### Convenience Accessors

The module pre-populates several module-level dictionaries and lists for common lookups:

- **`KafkaTopics`** — Maps `topic.requestedName` → `TopicConfig` object.
- **`ObjectBuckets`** — Maps `bucket.requestedName` → `ObjectStoreBucket` object.
- **`DependencyEndpoints`** — Nested dict: `[app_name][endpoint_name]` → `DependencyEndpoint` object.
- **`PrivateDependencyEndpoints`** — Nested dict: `[app_name][endpoint_name]` → `PrivateDependencyEndpoint` object.
- **`KafkaServers`** — List of `"hostname:port"` strings for all Kafka brokers.

These are built at import time by iterating over the loaded configuration. If the relevant section is missing or empty, the collection remains empty.

### Clowder Detection

The `isClowderEnabled()` function returns `True` if the `ACG_CONFIG` environment variable is set (to any non-empty value), `False` otherwise. This allows applications to conditionally enable Clowder integration.

## Build and Release

### Versioning Strategy

The library uses [setuptools_scm][setuptools-scm] for automatic versioning. Version numbers are derived from git tags:

- Tagged commits (e.g., `v1.2.3`) → version `1.2.3`
- Commits after a tag → version `1.2.3.devN+gHASH`

This eliminates manual version bumps. The [pyproject.toml][pyproject-file] declares `setuptools_scm[toml]` as a build dependency.

### CI/CD Pipeline

The [GitHub Actions workflow][ci-workflow] runs on pull requests, pushes to `master`, and release events:

1. **dist** — Builds source distribution and wheel using `python -m build`.
2. **test** — Installs the wheel and runs pytest across Python 3.10+.
3. **dist_check** — Validates distribution metadata with `twine check --strict`.
4. **dist_upload** — Publishes to PyPI on release events (requires `pypi_token` secret).

The test step sets `ACG_CONFIG=test.json`, which points to a [test fixture][test-json] containing sample Clowder configuration.

## Testing Approach

Tests verify the runtime API against a static JSON fixture ([test.json][test-json]). The test file includes:

- Sample Kafka broker/topic configuration
- Database credentials (including `rdsCa` for certificate testing)
- Object storage buckets
- Public and private dependency endpoints
- Feature flags and hostname

The [test suite][test-suite] validates:

- Correct deserialization of all config sections
- Population of convenience dictionaries (e.g., `KafkaTopics["originalName"].name == "someTopic"`)
- Temporary file creation for `rds_ca()` and `kafka_ca()`
- `isClowderEnabled()` returns `True` when `ACG_CONFIG` is set

There are no tests for the generated code itself (yacg's output is assumed correct). Tests focus on the handwritten wrapper logic in `__init__.py`.

## Key Design Decisions

### Schema-Driven vs. Hand-Written Models

**Decision:** Use code generation instead of hand-written Pydantic or dataclasses models.

**Rationale:**
- The Clowder schema is the authoritative source. Manual models would drift over time as new fields are added upstream.
- Regeneration is faster than manual updates across multiple fields/classes.
- Type safety is preserved without runtime validation overhead.

**Tradeoff:** Generated code includes generic docstrings (e.g., `"ClowdApp deployment configuration for Clowder enabled apps."` for every field). Hand-written models could provide better documentation.

### Eager vs. Lazy Configuration Loading

**Decision:** Load configuration at module import time (eager).

**Rationale:**
- Most applications need the config immediately on startup.
- Errors in the JSON file are detected early (before application logic runs).
- Simplifies API: users access `LoadedConfig` directly instead of calling a factory function.

**Tradeoff:** Import-time failures are harder to debug in some contexts (e.g., when importing the module in tests that don't set `ACG_CONFIG`).

### Temporary Files for CA Certificates

**Decision:** Write CA certificates to temporary files instead of requiring applications to do so.

**Rationale:**
- Most Python libraries (e.g., `psycopg2`, `kafka-python`) expect CA paths, not inline strings.
- The temporary file approach is more ergonomic than forcing every consumer to implement the same file-writing logic.
- Files are cached in `_rds_ca` and `_kafka_ca` attributes to avoid redundant writes.

**Tradeoff:** Files persist for the lifetime of the process (no cleanup on exit). This is acceptable for long-running services but could cause issues in short-lived CLI tools if called repeatedly.

### Single-File Generation

**Decision:** Generate all types into a single `types.py` file instead of one file per class.

**Rationale:**
- Simplifies the import structure (users import from `app_common_python.types`, not dozens of submodules).
- Matches the flat structure of the JSON schema (no nested modules in the schema definition).
- Easier to verify generation output (diff a single file instead of a directory tree).

**Tradeoff:** As the schema grows, the single `types.py` file will grow correspondingly, which could eventually impact IDE autocomplete performance.

## Future Considerations

- **Schema Validation:** The library does not validate JSON against the schema at runtime. Invalid JSON files produce AttributeErrors when accessing missing fields. Adding validation (e.g., with `jsonschema`) would improve error messages but increase dependencies.
- **Async Support:** The current API is synchronous. Applications using async frameworks (e.g., aiohttp, asyncpg) must wrap the config access in sync wrappers.
- **CLI for Schema Sync:** The `sync_config.sh` script could be replaced with a Python-based CLI (`python -m app_common_python.sync`) to avoid the Bash/Podman dependency.

[clowder-project]: https://github.com/RedHatInsights/clowder
[sync-script]: https://github.com/RedHatInsights/app-common-python/blob/master/sync_config.sh
[yacg-project]: https://github.com/OkieOth/yacg
[loadedconfig-module]: https://github.com/RedHatInsights/app-common-python/blob/master/app_common_python/__init__.py
[pyproject-file]: https://github.com/RedHatInsights/app-common-python/blob/master/pyproject.toml
[schema-file]: https://github.com/RedHatInsights/app-common-python/blob/master/schema.json
[types-file]: https://github.com/RedHatInsights/app-common-python/blob/master/app_common_python/types.py
[setuptools-scm]: https://github.com/pypa/setuptools_scm
[ci-workflow]: https://github.com/RedHatInsights/app-common-python/blob/master/.github/workflows/package.yml
[test-json]: https://github.com/RedHatInsights/app-common-python/blob/master/test.json
[test-suite]: https://github.com/RedHatInsights/app-common-python/blob/master/tests/test_config.py
