# FTP Listener Initialization in Ballerina Language Server

This document explains how the Ballerina Language Server (LS) exposes FTP Listener initialization configurations to the client.

## Overview

The FTP Listener initialization model is part of the **Service Model Generator** extension. It allows users to configure a service that monitors file actions (like adding or deleting files) on a remote FTP, SFTP, or FTPS server.

## API Endpoint

The configuration is retrieved using:

*   **Method**: `serviceDesign/getServiceInitModel`
*   **Request Parameter**: `ServiceModelRequest`
    *   Example: `{ "orgName": "ballerina", "packageName": "ftp", "moduleName": "ftp" }`

## Data Models

The LS uses the following primary models to structure the form:

### 1. `ServiceInitModel`
The root response containing metadata and a `properties` map.

### 2. `Value`
Individual configuration units. In FTP, these are often nested to create a hierarchical form using `GROUP_SECTION`, `CHOICE`, and `FORM` types.

## FTP Configuration Properties

The FTP initialization form is structured into two main sections: **Listener Configuration** and **Service Configuration**.

### 1. Listener Configuration (`configureListener`)
This is a `CHOICE` property that allows the user to either create a new listener or select an existing one.

#### Case A: Create New Listener
When "Create new" is selected, the following properties are exposed:

*   **`listenerVarName`**: `IDENTIFIER` field for the listener variable name (e.g., `ftpListener`).
*   **`designApproach` (Protocol)**: A `CHOICE` field to select the transfer protocol:
    *   **`ftp`**: Standard File Transfer Protocol.
    *   **`sftp`**: SSH File Transfer Protocol.
    *   **`ftps`**: FTP over SSL/TLS.

Common fields within each protocol:
*   **`host`**: `TEXT` field for the remote server address.
*   **`portNumber`**: `NUMBER` field for the connection port (default is 21).
*   **`authentication`**: A protocol-specific `CHOICE` field:
    *   **FTP/FTPS**: "No Authentication" or "Basic Authentication" (Username/Password).
    *   **SFTP**: "No Authentication" or "Certificate Based Authentication" (`privateKey` and `userName`).
*   **`secureSocket`** (FTPS only): `RECORD_MAP_EXPRESSION` for `ftp:SecureSocket`.

#### Case B: Use Existing Listener
If compatible listeners are found in the project, the "Use existing" branch is enabled:
*   **`existingListener`**: A `SINGLE_SELECT` dropdown listing available listener names.
*   Selecting a listener shows its current configuration in a **read-only** view.

### 2. Service Configuration (`path`)
*   **Monitoring Path**: A `TEXT` field defining the folder on the remote server to monitor (e.g., `"/uploads"`). This value is mapped to the `@ftp:ServiceConfig` annotation.

## Key Classes and Responsibilities

| Class | Responsibility |
| :--- | :--- |
| `ServiceModelGeneratorService` | The entry point for the `serviceDesign/getServiceInitModel` LSP endpoint. |
| `ServiceBuilderRouter` | Routes the incoming `ServiceModelRequest` to the `FTPServiceBuilder`. |
| `FTPServiceBuilder` | Orchestrates the model generation. It loads the base structure from `ftp_init.json`, filters legacy listeners, and builds the final `ServiceInitModel`. |
| `FTPListenerUtil` | Specialized utility that parses existing `ftp:Listener` declarations from the syntax tree to extract their configuration (protocol, host, auth, etc.). |
| `FTPFunctionModelUtil` | Handles function-level configuration, specifically the `@ftp:FunctionConfig` annotation and its post-process actions (`onSuccess`, `onError`). |
| `ListenerUtil` | Provides project-wide scanning utilities to find existing listeners compatible with the `ftp` module. |

## Implementation Details & Expert Thoughts

### 1. Protocol-Specific Authentication
The FTP model is more complex than standard JMS models because the authentication options change based on the selected protocol.
- **Expert Note**: When adding a new authentication method, you must update both the `ftp_init.json` template and the `FTPListenerUtil.buildAuthChoiceValue` logic to ensure existing listeners can still be correctly parsed and displayed.

### 2. Legacy vs. New Pattern
The LS distinguishes between "Legacy" FTP listeners (which often included the `path` directly in the listener constructor) and the "New" pattern (where `path` is in `@ftp:ServiceConfig`).
- **Expert Note**: `FTPServiceBuilder.filterNonLegacyListeners` ensures that only listeners compatible with the new annotation-based pattern are offered in the "Use existing" dropdown. This is a critical safety check to prevent users from accidentally mixing patterns that might lead to runtime errors.

### 3. Metadata Merging for Existing Listeners
When displaying an existing listener, the LS doesn't just show the raw source values. It extracts the values using `FTPListenerUtil` and then applies the labels and descriptions from the `ftp_init.json` template using `FTPServiceBuilder.applyInitModelMetadata`.
- **Expert Note**: This ensures a consistent UI experience. If you change a label in `ftp_init.json`, it will automatically update for both the "Create new" and "Use existing" views.

### 4. Code Generation Logic
The code generation in `addServiceInitSource` manually constructs the `listener ftp:Listener ... = new(...)` string.
- **Expert Note**: Pay close attention to the `auth` mapping construction. It handles nested records for `credentials` (in Basic Auth) and `privateKey` (in SFTP). Any change to the connector's `auth` record structure requires a corresponding update here.

### 5. Service-Level vs. Function-Level Annotations
- **`@ftp:ServiceConfig`**: Managed by `FTPServiceBuilder`. Controls the global monitoring `path`.
- **`@ftp:FunctionConfig`**: Managed by `FTPFunctionModelUtil`. Controls post-process actions like `DELETE` or `MOVE` after a file is processed.

## Key Areas for Careful Consideration

1.  **`ftp_init.json` Structure**: This file is the "source of truth" for the UI structure. It uses `GROUP_SECTION` to organize fields. If you modify the nesting, ensure the unwrap logic in `FTPServiceBuilder.addServiceInitSource` still works.
2.  **`FTPListenerUtil` Parsing**: The parser is sensitive to the syntax of the listener declaration. It handles `NamedArgumentNode` and looks for specific keys like `protocol`, `host`, and `auth`. If a new parameter is added to the Ballerina `ftp:Listener` init method, it must be explicitly handled in `extractConfigFromListenerDeclaration`.
3.  **SFTP Private Key**: The `privateKey` is a `RECORD_MAP_EXPRESSION` (of type `ftp:PrivateKey`). Ensure that the `typeMembers` in the JSON or the `buildReadOnlyRecordValue` method correctly point to the right package version.
4.  **Backward Compatibility**: The builder still contains code to handle legacy payloads (e.g., checking for `folderPath` if `path` is missing). Maintain these checks to avoid breaking older LS clients.
