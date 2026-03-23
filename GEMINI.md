# Ballerina Language Server - Project Context

## Project Overview
The Ballerina Language Server is a Java-based implementation of the [Language Server Protocol (LSP)](https://microsoft.github.io/language-server-protocol/) for the [Ballerina](https://ballerina.io/) programming language. It enables IDEs to provide rich language features such as auto-completion, diagnostics, go-to-definition, and refactoring for Ballerina.

### Main Technologies
*   **Java**: Primary programming language.
*   **Gradle**: Build and dependency management.
*   **LSP4J**: Eclipse library for LSP implementation.
*   **Ballerina SDK**: Used for language analysis and compilation.

### Architecture
The project is highly modular:
*   **`langserver-core`**: The heart of the language server, containing LSP method implementations (completion, hover, diagnostics, etc.).
*   **`langserver-commons`**: Common utilities and interfaces used across modules.
*   **`langserver-stdlib`**: Standard library support for the language server.
*   **`launchers`**: Entry points for different transport protocols (e.g., `stdio-launcher` for standard I/O).
*   **Generators & Extensions**: Specialized modules for model generation (Architecture, Flow, Sequence, etc.) and custom LS extensions.

## Building and Running

### Key Commands
*   **Build the entire project**:
    ```bash
    ./gradlew build
    ```
*   **Create a fat/shadow JAR (All modules included)**:
    ```bash
    ./gradlew pack
    ```
    The output JAR will be located at `build/ballerina-language-server-<version>.jar`.
*   **Run the Language Server (via launcher)**:
    ```bash
    ./gradlew :launcher:run
    ```
*   **Run Tests**:
    ```bash
    ./gradlew test
    ```
    *Note: The test task automatically handles unpacking the required Ballerina distribution for testing.*

### Main Entry Point
The primary entry point for the standard I/O launcher is:
`org.ballerinalang.langserver.launchers.stdio.Main`

## Development Conventions

### Coding Standards
*   **Checkstyle**: Enforced via the `checkstyle` module and Gradle plugin. Configuration is in `build-config/checkstyle`.
*   **Spotbugs**: Used for static analysis to find bugs. Exclusions are managed in `spotbugs-exclude.xml`.
*   **Formatting**: Adhere to existing codebase patterns (standard Java conventions with WSO2/Ballerina specific headers).

### Testing Practices
*   **Mandatory Testing**: Every feature or bug fix must include automated tests.
*   **AbstractLSTest**: Most tests in `langserver-core` extend `AbstractLSTest` for standardized LSP interaction testing.
*   **Test Data**: JSON-based expectations and Ballerina source files are commonly used for integration testing.

### Contribution Guidelines
*   **PR Template**: Follow the `pull_request_template.md` which requires documentation of Purpose, Approach, User Stories, and Security Checks.
*   **Security**: Use the FindSecurityBugs plugin and follow secure coding standards. Never commit secrets.
