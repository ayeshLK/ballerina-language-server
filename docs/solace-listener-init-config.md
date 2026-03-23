# Solace Listener Initialization in Ballerina Language Server

This document explains how the Ballerina Language Server (LS) exposes Solace Listener initialization configurations to the client (typically a UI form).

## Overview

The Language Server provides a specialized endpoint to retrieve the metadata and property configurations required to create a Solace service. This is part of the **Service Model Generator** extension, which dynamically constructs form-based UI models for various Ballerina connectors.

## API Endpoint

The configuration is exposed via the following LSP JSON-RPC request:

*   **Method**: `serviceDesign/getServiceInitModel`
*   **Request Parameter**: `ServiceModelRequest`
    *   Contains information like the organization name (`ballerinax`), package name (`solace`), and module name (`solace`).

## Data Models

The following models represent the structure of the initialization form.

### 1. `ServiceInitModel`
This is the root model returned by the API. It contains general metadata about the connector and a map of properties that define the form fields.

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `String` | Unique identifier for the service type. |
| `displayName` | `String` | The human-readable name (e.g., "Solace Event Integration"). |
| `description` | `String` | A brief summary of what the service does. |
| `properties` | `Map<String, Value>` | A key-value map where each `Value` represents a form field. |

### 2. `Value`
Each entry in the `properties` map is a `Value` object. This model tells the UI how to render an input field, what its default value is, and whether it's editable.

*   **`metadata`**: Contains the `label` and `description` for the UI field.
*   **`value`**: The current or default value of the field.
*   **`fieldType`**: (Nested in `PropertyType`) Determines the UI component to use (e.g., `TEXT`, `SINGLE_SELECT`, `CHOICE`).
*   **`enabled` / `editable`**: Boolean flags to control UI state.
*   **`choices` / `options`**: Available options for selection fields.

## Solace Configuration Properties

When `getServiceInitModel` is called for Solace, the `SolaceServiceBuilder` class populates the properties. However, the structure of these properties changes depending on whether any Solace listeners already exist in the project.

### Case 1: No Existing Listeners
If no compatible Solace listeners are found, the properties are listed directly in the `properties` map.

*   **`url`**: `TEXT` field for broker URL.
*   **`authentication`**: `CHOICE` field for auth methods.
*   **`secureSocket`**: `CHOICE` field for TLS.
*   **`destination`**: `CHOICE` field for Queue/Topic.
*   **`sessionAckMode`**: `SINGLE_SELECT` field for acknowledgment mode.

### Case 2: Existing Listeners Found
If one or more Solace listeners exist (e.g., `solace:Listener solaceListener = ...`), the LS introduces a **`configureListener`** property. This property is a `CHOICE` type that forces the user to choose between using an existing listener or creating a new one.

When this happens:
1.  Listener-specific properties (`url`, `authentication`, `secureSocket`) are **removed** from the top-level `properties` map.
2.  They are **moved** into the "Create New Listener" branch of the `configureListener` branch.
3.  A new branch "Use Existing Listener" is added, containing a dropdown of available listeners.

## Key Classes and Responsibilities

| Class | Responsibility |
| :--- | :--- |
| `ServiceModelGeneratorService` | The LSP service entry point that handles the `serviceDesign/getServiceInitModel` request. |
| `ServiceBuilderRouter` | Identifies the correct builder (e.g., `SolaceServiceBuilder`) based on the module name in the request. |
| `SolaceServiceBuilder` | Orchestrates the creation of the Solace-specific `ServiceInitModel`. It handles the high-level logic of property removal and relocation when existing listeners are found. |
| `JmsUtil` | Provides reusable logic for building complex JMS-related properties such as `Authentication`, `SecureSocket`, `Destination`, and `SessionAckMode`. |
| `ListenerUtil` | Provides generic utilities for scanning the project for compatible listeners and building the "Use Existing" vs. "Create New" `CHOICE` property. |
| `ServiceInitModel` | The top-level data transfer object (DTO) that holds the metadata and the map of form properties. |
| `Value` | The DTO representing an individual form field, its metadata, type, and current value. |

## Implementation Details & Expert Thoughts

### 1. Property Mapping (UI vs. Code)
The LS uses specific property keys in the UI that might differ from the actual Ballerina listener or service configuration keys.
- **`authentication`** in UI is transformed into **`auth`** in the generated Ballerina code. The `SolaceServiceBuilder.applyAuthenticationProperty` method handles this transformation, converting a complex choice into a record-like expression string.
- **`destination`** is a `CHOICE` between Queue and Topic. This choice is flattened during code generation to extract `queueName` or `topicName`.

### 2. Shared JMS Utilities
`JmsUtil` is a shared utility class. While it helps maintain consistency across different JMS connectors (Solace, RabbitMQ, etc.), developers must be careful when modifying it as it can have side effects on other connectors.
- **Acknowledgment Mode**: The `sessionAckMode` affects the `onMessage` function signature. If `CLIENT_ACKNOWLEDGE` is selected, a `solace:Caller` parameter is added to the function.

### 3. Dependency on External Metadata
Base properties like `url` and `messageVpn` are not hardcoded in the Java classes. They are retrieved from the `ServiceDatabaseManager`, which loads metadata from the connector's compiled package. Any changes to the connector's `init` function parameters will be automatically reflected here if the database is updated.

### 4. Advanced Properties
Properties like `secureSocket` and `authentication` are marked as **Advanced**. UI implementations typically hide these behind an "Advanced" toggle to keep the initial form simple.

## Key Areas for Careful Consideration

1.  **`LISTENER_CONFIG_KEYS`**: This list in `SolaceServiceBuilder` determines which properties are moved under the "Create New Listener" branch when existing listeners are found. If a new listener property is added (e.g., a new timeout or connection setting), it **must** be added to this list.
2.  **`buildAuthenticationConfig`**: This method manually constructs the record string for the `auth` property. If a new authentication method or property is added to `JmsUtil.buildAuthenticationChoice`, this method must be updated to ensure it's correctly serialized.
3.  **Service Annotation Generation**: The `JmsUtil.buildServiceAnnotation` method uses a `consumerTypeResolver` to determine the `consumerType` (e.g., `DURABLE` vs `DEFAULT`). Ensure this logic aligns with the latest Solace connector specifications.
4.  **Function Data Binding**: The `SolaceServiceBuilder.getModelFromSource` and `SolaceFunctionBuilder` handle data binding for the `onMessage` function. If the payload type mapping changes, these classes need updates.
