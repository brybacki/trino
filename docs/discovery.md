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

1.  **Embedded Discovery Server:**
    *   In this mode, one or more coordinators also run an embedded Airlift discovery server.
    *   The `CoordinatorDiscoveryModule.java` is responsible for installing Airlift's `EmbeddedDiscoveryModule` when `EmbeddedDiscoveryConfig` is enabled.
    *   All nodes in the cluster (workers and other coordinators) are configured to register with and query this embedded server running on the coordinator(s).

2.  **External Discovery Service:**
    *   Trino nodes can be configured to use an external, standalone discovery service, such as HashiCorp Consul.
    *   All nodes (coordinators and workers) register themselves with this external service and query it to find other nodes.

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
