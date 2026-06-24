# app-common-python

Python client library for accessing [Clowder operator][clowder-operator] configuration in Red Hat Insights applications. The library provides a simple API for reading environment-based configuration and accessing Kafka, database, object storage, and service dependency settings.

## Installation

Install from PyPI:

```sh
pip install app-common-python
```

## Prerequisites

- Python 3.10+
- Applications running in a Clowder-enabled environment with the `ACG_CONFIG` environment variable set

## Usage

### Basic Configuration Access

The library loads configuration automatically from the `ACG_CONFIG` environment variable when imported. Use `isClowderEnabled()` to detect whether Clowder is active, then access settings via the `LoadedConfig` object:

```python
from app_common_python import LoadedConfig, isClowderEnabled

def main():
    if isClowderEnabled():
        print(f"Public Port: {LoadedConfig.publicPort}")
        print(f"Hostname: {LoadedConfig.hostname}")
```

### API Overview

The library provides several convenience accessors for common configuration lookups:

#### Certificate Helpers

- **`LoadedConfig.rds_ca()`** — Creates a temporary file containing the RDS CA certificate and returns the file path. Useful for database libraries that require CA certificates as file paths.
- **`LoadedConfig.kafka_ca()`** — Creates a temporary file containing the Kafka CA certificate from the first broker and returns the file path.

#### Kafka Configuration

- **`KafkaTopics`** — Dictionary mapping `requestedName` (your application's topic name) to `TopicConfig` objects containing the actual Kafka topic name and configuration.
- **`KafkaServers`** — List of Kafka broker URLs in `hostname:port` format.

Example:

```python
from app_common_python import KafkaServers, KafkaTopics

# Connect to Kafka brokers
brokers = KafkaServers  # ["kafka-broker-1:9092", "kafka-broker-2:9092"]

# Look up the actual topic name for a requested topic
topic_config = KafkaTopics["my-topic"]
actual_topic_name = topic_config.name
```

#### Object Storage

- **`ObjectBuckets`** — Dictionary mapping `requestedName` to `ObjectStoreBucket` objects containing bucket name, access credentials, and endpoint information.

#### Service Dependencies

- **`DependencyEndpoints`** — Nested dictionary `[app_name][deployment_name]` → `DependencyEndpoint` for accessing public service endpoints of other applications.
- **`PrivateDependencyEndpoints`** — Nested dictionary `[app_name][deployment_name]` → `PrivateDependencyEndpoint` for accessing private service endpoints.

Example:

```python
from app_common_python import DependencyEndpoints

# Access another service's endpoint
other_service = DependencyEndpoints["other-app"]["api"]
service_url = f"http://{other_service.hostname}:{other_service.port}"
```

For a detailed explanation of the library's schema-driven design and code generation pipeline, see the [Architecture documentation][architecture-doc].

## Development

### Local Setup

Clone the repository and create a virtual environment:

```sh
git clone https://github.com/RedHatInsights/app-common-python.git
cd app-common-python
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -e .
```

### Running Tests

Tests use a static configuration file (`test.json`) to verify the runtime API. Run the test suite with:

```sh
ACG_CONFIG=test.json pytest
```

### Regenerating Types

The library uses code generation to stay in sync with the upstream Clowder schema. To regenerate the `types.py` file after schema changes:

```sh
./sync_config.sh
```

This script requires Podman and fetches the latest schema from the Clowder project, then regenerates the Python types using the [yacg code generator][yacg-project].

### Linting

The project uses flake8 for code style checks:

```sh
flake8 app_common_python tests
```

Configuration is in `setup.cfg` (max line length: 100, ignores: E128, E811, W503, E203).

### Contributing

See the [Contributing guidelines][contributing-doc] for information on commit signing, pull request workflow, and code review process.

## License

**License information not found in repository.** Contributors should clarify licensing with project maintainers before submitting changes.

## Related Documentation

- [Architecture][architecture-doc] — Schema-driven design and code generation pipeline
- [Contributing][contributing-doc] — Commit signing, pull request workflow, and guidelines
- [Clowder Operator][clowder-operator] — Upstream Kubernetes operator project

[clowder-operator]: https://github.com/RedHatInsights/clowder
[architecture-doc]: ./ARCHITECTURE.md
[contributing-doc]: ./CONTRIBUTING.md
[yacg-project]: https://github.com/OkieOth/yacg
