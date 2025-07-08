# Trino and Airlift Discovery

Trino leverages the Airlift Discovery library (`io.airlift.discovery`) for service discovery, enabling coordinators to dynamically find and manage worker nodes within a cluster. This document outlines how this mechanism works.

## Service Announcement

All Trino nodes, whether they are coordinators or workers, participate in the discovery process by announcing their presence.

1.  **Service Type:** Each Trino node announces itself under a common service type, typically `"trino"`. This allows discovery clients to find all potential nodes in the cluster.
2.  **Announcement Properties:** Along with the service type, each node publishes key-value properties, including:
    *   `node_version`: The version of the Trino software the node is running.
    *   `http` / `https`: The HTTP or HTTPS URI where the node can be reached.
    *   `coordinator`: A boolean flag (`true` or `false`) indicating whether the node is a coordinator.
    *   `catalogHandleIds` (optional): A comma-separated list of catalog handles that the node supports, used when not all catalogs are available on all nodes.

This announcement is configured in `ServerMainModule.java` using `discoveryBinder(binder).bindHttpAnnouncement("trino")`.

## Discovery Server Options

Trino supports two primary modes for the discovery server backend:

### 1. Embedded Discovery Server

In this mode, one or more coordinators also run an embedded Airlift discovery server.
The `CoordinatorDiscoveryModule.java` is responsible for installing Airlift's `EmbeddedDiscoveryModule` when `EmbeddedDiscoveryConfig` is enabled. All nodes in the cluster (workers and other coordinators) are configured to register with and query this embedded server running on the coordinator(s).

**Flow:**
```
+-----------------+      Announces (self)      +--------------------------+
| Coordinator X   |<---------------------------| EmbeddedDiscoveryModule  |
| (runs           |                            | (runs on Coordinator X)  |
| DiscoveryNodeManager)|----Queries Services--->| Stores:                  |
+-----------------+                            |  - Coord X (properties)  |
      ^                                        |  - Worker A (properties) |
      | Announces                              |  - Worker B (properties) |
      |                                        +--------------------------+
      |                                                ^      ^
+-----------------+                                  |      | Announces
| Worker A        |------------ Announces -----------+      |
+-----------------+                                         |
                                                            |
+-----------------+                                         |
| Worker B        |------------------ Announces -----------+
+-----------------+
```

#### Understanding `EmbeddedDiscoveryModule`

The `io.airlift.discovery.server.EmbeddedDiscoveryModule` is a Guice module provided by Airlift that allows an application to host a lightweight, in-process discovery server. Here's how it generally works:

*   **In-Process & In-Memory:** It runs within the same Java process as the Trino coordinator. Service announcements are stored in memory (typically in simple map-like structures). This makes it very fast for lookups.
*   **Guice Integration:** As a Guice module, it sets up the necessary bindings for the discovery server components. When `CoordinatorDiscoveryModule` installs it, these components become part of the coordinator's runtime environment.
*   **JAX-RS Resources:** It exposes standard JAX-RS (Java API for RESTful Web Services) resource endpoints. These are the HTTP endpoints that discovery clients use to:
    *   **Announce:** A node sends a PUT request with its service information (type, pool, properties) to an endpoint like `/v1/announcement/{node_id}`. The `EmbeddedDiscoveryModule` then stores or updates this information in its in-memory store.
    *   **Unannounce:** A node sends a DELETE request to the same endpoint to remove its registration.
    *   **Fetch/Query:** Clients (like Trino's `DiscoveryNodeManager` via `ServiceSelector`) send GET requests to endpoints like `/v1/service/{service_type}/{pool}` or `/v1/service/{service_type}` to retrieve a list of currently registered services and their properties.
*   **Simplicity:** Its main advantage is deployment simplicity. It avoids the need for setting up and maintaining a separate, external discovery system (like Consul or ZooKeeper). This makes it ideal for:
    *   Single-coordinator Trino deployments.
    *   Development and testing environments.
    *   Smaller clusters where the operational overhead of an external system is not justified.
*   **No Persistence (Typically):** Because it's in-memory, service registrations are usually lost if the coordinator process restarts. Nodes would then need to re-announce themselves.
*   **Conceptual View:**
    ```
    +--------------------------+
    | Coordinator Process      |
    |                          |
    |  +---------------------+ |
    |  | EmbeddedDiscovery   | |
    |  | Module              | |
    |  |---------------------| |
    |  | - In-memory store   | |
    |  |   (Map of services) | |
    |  | - Handles Announce  | |
    |  |   (add/update map)  | |
    |  | - Handles Query     | |
    |  |   (read from map)   | |
    |  +---------------------+ |
    |                          |
    |  +---------------------+ |
    |  | DiscoveryNodeManager| |
    |  | (ServiceSelector)   | |
    |  |  |                  | |
    |  |  `-- Queries self   | |
    |  +---------------------+ |
    |                          |
    +--------------------------+
    ```

In a Trino context, if `EmbeddedDiscoveryConfig` enables this module, the coordinator essentially becomes the discovery hub for the cluster.

### 2. External Discovery Service

Trino nodes can be configured to use an external, standalone discovery service, such as HashiCorp Consul or etcd. All nodes (coordinators and workers) register themselves with this external service and query it to find other nodes.

**Flow:**
```
+-----------------+      Queries Services      +--------------------------+
| Coordinator X   |<---------------------------| External Discovery       |
| (runs           |                            | Service (e.g., Consul)   |
| DiscoveryNodeManager)|                            |                          |
+-----------------+----Announces------------------>| Stores:                  |
      ^                                        |  - Coord X (properties)  |
      | Announces                              |  - Worker A (properties) |
      |                                        |  - Worker B (properties) |
+-----------------+                            +--------------------------+
| Worker A        |----- Announces ----------------->|          ^
+-----------------+                                            |
                                                               |
+-----------------+                                            |
| Worker B        |---------- Announces ---------------------->|
+-----------------+
```
This mode is generally preferred for larger, production, or High Availability (HA) Trino clusters as it decouples the discovery mechanism from individual coordinator nodes.

## Coordinator's Role in Discovery

The Trino coordinator plays a central role in utilizing the discovery information:

1.  **Finding Nodes:** The coordinator uses an Airlift `ServiceSelector` to query the discovery server (embedded or external) for all registered services of type `"trino"`.
2.  **Node Management (`DiscoveryNodeManager`):**
    *   The `DiscoveryNodeManager` class is responsible for maintaining the state of all nodes in the cluster.
    *   It periodically fetches the list of registered "trino" services.
    *   For each discovered node, it polls a status endpoint (commonly `/v1/info/state`) to determine its current state (e.g., `ACTIVE`, `INACTIVE`, `DRAINING`, `SHUTTING_DOWN`).
    *   It uses the `coordinator` property from the service announcement to differentiate between worker nodes and other coordinator nodes.
3.  **Cluster View:** Based on this information, the coordinator builds a comprehensive view of the cluster, including active workers available for task execution and the status of other coordinators (if any).

## Worker's Role in Discovery

Workers primarily act as announcers in this discovery process:

1.  **Announcement:** Workers announce their presence and properties to the configured discovery server (embedded or external).
2.  **Passive Discovery:** Workers generally do not need to discover other workers directly for query execution. Task scheduling and distribution are managed by the coordinator, which already has the cluster view from the discovery service.

## Summary

Airlift Discovery provides a robust and flexible mechanism for Trino cluster membership and state management. The coordinator uses it to discover, monitor, and manage worker nodes, ensuring efficient task scheduling and fault tolerance. Workers, in turn, ensure they are discoverable by the coordinator by announcing their availability. This system allows Trino clusters to scale dynamically and adapt to changes in node availability.
The choice between an embedded or external discovery server depends on the scale, complexity, and resilience requirements of the Trino deployment.
