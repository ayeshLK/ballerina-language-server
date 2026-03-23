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
Individual configuration units. In FTP, these are often nested to create a hierarchical form.

## FTP Configuration Properties

The FTP initialization form is structured into two main sections: **Listener Configuration** and **Service Configuration**.

### 1. Listener Configuration (`configureListener`)
This is a `CHOICE` property that allows the user to either create a new listener or select an existing one.

#### Case A: Create New Listener
When "Create new" is selected, the following properties are exposed:

*   **`listenerVarName`**: `IDENTIFIER` field for the listener variable name (e.g., `ftpListener`).
*   **`designApproach` (Protocol)**: A `CHOICE` field to select the transfer protocol:
    *   **`ftp`**: Standard File Transfer Protocol.
    *   **`sftp`**: SSH File Transfer Protocol (supports Certificate-based auth).
    *   **`ftps`**: FTP over SSL/TLS (supports `secureSocket` configuration).

Common fields within each protocol:
*   **`host`**: `TEXT` field for the remote server address.
*   **`portNumber`**: `NUMBER` field for the connection port (default is 21).
*   **`authentication`**: `CHOICE` field for "No Authentication" or "Basic Authentication" (Username/Password).

#### Case B: Use Existing Listener
If compatible listeners are found in the project, this branch is enabled:
*   **`existingListener`**: A `SINGLE_SELECT` dropdown listing available listener names.

### 2. Service Configuration (`path`)
*   **Monitoring Path**: A `TEXT` field defining the folder on the remote server to monitor (e.g., `"/uploads"`). This value is mapped to the `@ftp:ServiceConfig` annotation.

## JSON Samples

### Case A: Create New Listener (Standard Model)

This sample shows the properties map when no existing listeners are found or when "Create new" is selected.

```json
{
  "properties": {
    "configureListener": {
      "metadata": { "label": "Configure FTP Listener" },
      "value": "0",
      "types": [{ "fieldType": "CHOICE" }],
      "choices": [
        {
          "metadata": { "label": "Create new" },
          "properties": {
            "listenerConfig": {
              "types": [{ "fieldType": "GROUP_SECTION" }],
              "properties": {
                "listenerVarName": { "value": "ftpListener", "types": [{ "fieldType": "IDENTIFIER" }] },
                "designApproach": {
                  "metadata": { "label": "Protocol" },
                  "types": [{ "fieldType": "CHOICE" }],
                  "choices": [
                    {
                      "metadata": { "label": "ftp" },
                      "properties": {
                        "host": { "value": "\"127.0.0.1\"", "types": [{ "fieldType": "TEXT" }] },
                        "portNumber": { "value": "21", "types": [{ "fieldType": "NUMBER" }] },
                        "authentication": { "types": [{ "fieldType": "CHOICE" }] }
                      }
                    }
                  ]
                }
              }
            }
          }
        }
      ]
    },
    "path": {
      "metadata": { "label": "Monitoring Path" },
      "value": "\"/\"",
      "types": [{ "fieldType": "TEXT" }]
    }
  }
}
```

### Case B: Use Existing Listener Model

This sample shows how the `configureListener` property is updated when existing listeners (e.g., `ftpListener1`) are detected.

```json
{
  "properties": {
    "configureListener": {
      "metadata": { "label": "Configure FTP Listener" },
      "value": "1",
      "types": [{ "fieldType": "CHOICE" }],
      "choices": [
        {
          "metadata": { "label": "Create new" },
          "enabled": false
        },
        {
          "metadata": { "label": "Use existing" },
          "enabled": true,
          "properties": {
            "listenerConfig": {
              "types": [{ "fieldType": "GROUP_SECTION" }],
              "properties": {
                "existingListener": {
                  "metadata": { "label": "Select Listener" },
                  "value": "ftpListener1",
                  "types": [
                    {
                      "fieldType": "SINGLE_SELECT",
                      "options": [
                        { "label": "ftpListener1", "value": "ftpListener1" }
                      ]
                    }
                  ]
                }
              }
            }
          }
        }
      ]
    },
    "path": {
      "metadata": { "label": "Monitoring Path" },
      "value": "\"/\"",
      "types": [{ "fieldType": "TEXT" }]
    }
  }
}
```

## Key Classes and Responsibilities

The following classes are responsible for generating the FTP initialization configuration:

| Class | Responsibility |
| :--- | :--- |
| `ServiceModelGeneratorService` | The entry point for the `serviceDesign/getServiceInitModel` LSP endpoint. |
| `ServiceBuilderRouter` | Routes the incoming `ServiceModelRequest` to the `FTPServiceBuilder`. |
| `FTPServiceBuilder` | Loads the base model from `ftp_init.json`, identifies existing listeners, and populates metadata for selected listener properties. |
| `FTPListenerUtil` | Specialized utility that extracts configuration from existing `ftp:Listener` declarations in the project's source code. |
| `ListenerUtil` | Provides project-wide scanning utilities to find existing listeners that are compatible with the `ftp` module. |
| `ServiceInitModel` | The root data structure that represents the entire initialization form. |
| `Value` | The field-level model that provides metadata (label, description) and UI hints (`fieldType`). |

## Processing Flow

1.  **Request**: Client requests the model for `ballerina/ftp`.
2.  **Resource Loading**: `FTPServiceBuilder` loads the base structure from `ftp_init.json`.
3.  **Discovery**:
    *   The LS identifies compatible listeners using `ListenerUtil`.
    *   If found, it populates the "Use existing" choice and sets it as the default.
4.  **Metadata Merging**: For existing listeners, the LS extracts their current configurations and merges them with the UI metadata (labels/descriptions) from the template to ensure a consistent look.
5.  **Response**: The final `ServiceInitModel` is returned to the client.
