app-common-python
=================

Simple client access library for the config for the Clowder operator.

Usage
-----

To access configuration, see the following example

```python
from app_common_python import LoadedConfig, isClowderEnabled

def main():
    if isClowderEnabled():
        print(f"Public Port: {LoadedConfig.PublicPort}")
```

The ``clowder`` library also comes with several other helpers

* ``LoadedConfig.rds_ca()`` - creates a temporary file with the RDSCa and
  returns the filename.
* ``LoadedConfig.kafka_ca()`` - creates a temporary file with the KafkaCa and
  returns the filename from the frist broker in the list.
* ``KafkaTopics`` - returns a map of KafkaTopics using the requestedName
  as the key and the topic object as the value.
* ``KafkaServers`` - returns a list of Kafka Broker URLs.
* ``ObjectBuckets`` - returns a list of ObjectBuckets using the requestedName
  as the key and the bucket object as the value.
* ``DependencyEndpoints`` - returns a nested map using \[appName\]\[deploymentName\]
  for the public services of requested applications.
* ``PrivateDependencyEndpoints`` - returns a nested map using \[appName\]\[deploymentName\]
  for the private services of requested applications.
* ``DependencyEndpointsV2`` - returns a nested map using \[appName\]\[endpointName\]
  for V2 dependency endpoints with simplified connection info.
* ``get_v2_dependency_endpoint(app, endpoint)`` - returns a single V2 endpoint
  or ``None`` if not found.

V2 Dependency Endpoints
-----------------------

V2 endpoints provide simplified, opinionated configuration for service-to-service
connections. Each endpoint includes a single URI, an authentication flag, and an
optional CA certificate path.

```python
from app_common_python import DependencyEndpointsV2, get_v2_dependency_endpoint, isClowderEnabled

if isClowderEnabled():
    # Access via the nested map
    rbac = DependencyEndpointsV2["rbac"]["service"]
    print(f"URI: {rbac.uri}")
    print(f"Authenticated: {rbac.authenticated}")
    if rbac.ca_certificate:
        print(f"CA Cert: {rbac.ca_certificate}")

    # Or use the helper function
    ep = get_v2_dependency_endpoint("rbac", "service")
    if ep:
        print(f"URI: {ep.uri}")
```

V2 endpoint fields:
* ``uri`` - Full URI for the service (e.g. ``https://rbac-service.env.svc:8443``)
* ``authenticated`` - ``True`` for cross-cluster (ClowdAppRef), ``False`` for in-cluster
* ``ca_certificate`` - Path to CA certificate for TLS (only present for in-cluster TLS)

Testing
-------

``ACG_CONFIG="test.json" pytest``
