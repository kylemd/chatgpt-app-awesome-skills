Based on the research and documentation provided in the uploaded files, here is a comprehensive specification of the ChatGPT Code Execution Sandbox (often referred to as "ChatGPT Containers").
> Credit & Attribution:
> The findings below regarding network behavior, environment variables, and multi-language support are derived from research by Simon Willison.
>  * Source: Simon Willison’s Weblog - ChatGPT Containers
>  * Technical Notes: chatgpt-container-environment_README.md
>  * Tool Definitions: available_tools_RESPONSE.md (User provided tool dump)
> 
ChatGPT Container Environment Specification
1. System Overview & Architecture
The environment is a sandboxed, ephemeral container designed for secure code execution and file manipulation. It operates on a "Split Egress" security model, where direct internet access from user code is blocked, but specific privileged pathways are whitelisted for package management and file retrieval.
Operating System
 * OS: Linux-based container (likely Debian/Ubuntu based on standard paths).
 * User Context: Code runs as a restricted user (often oai or sandbox), but specific tools like npm may see root owned paths in the file system.
Network Security Model
The environment enforces a strict "packages-only" egress policy via environment variables like NETWORK=caas_packages_only.
| Access Type | Status | Mechanism |
|---|---|---|
| General Internet | Blocked | urllib, requests, fetch, curl to public IPs fail with [Errno 101] Network is unreachable. |
| Public File Download | Allowed | Must be performed via the privileged container.download tool, not user code. |
| Package Managers | Allowed | pip, npm, uv, and apk are routed through an internal proxy (packages.applied-caas-gateway1.internal.api.openai.org). |
| Git/GitHub | Blocked | git clone and GitHub zip downloads fail; users must install via PyPI/NPM or use pip download for source tarballs. |
2. Runtime & Language Support
While originally limited to Python, the container now supports a wide array of languages executable via the container.exec (Bash) interface.
Primary Runtimes
 * Python: Full support via python.exec (internal reasoning) and python_user_visible.exec (charts/plots).
 * Node.js: Full support via node and npm.
 * Bash: Direct shell execution via container.exec.
Additional Compiled/Interpreted Languages
The following languages have been confirmed as installed and executable via container.exec (Bash):
 * C / C++ (gcc, g++)
 * Go
 * Java
 * Ruby
 * PHP
 * Perl
 * Swift
 * Kotlin
(Note: Rust and Docker are not currently installed, though environment variables suggest future or internal support for their registries.)
3. Package Management & Proxies
To allow users to install libraries without granting full internet access, the environment forces package managers to use an internal mirror.
Python (pip, uv)
 * Mirror URL: https://reader:****@packages.applied-caas-gateway1.internal.api.openai.org/.../pypi-public/simple
 * Configuration: Managed via PIP_INDEX_URL, PIP_TRUSTED_HOST, UV_INDEX_URL.
 * Behavior: Users can run pip install <package> successfully. Metadata inspection (importlib.metadata) works as expected.
Node.js (npm)
 * Mirror URL: https://reader:****@packages.applied-caas-gateway1.internal.api.openai.org/.../npm-public
 * Configuration: Managed via NPM_CONFIG_REGISTRY.
Docker (Registry Only)
 * Status: The Docker CLI is missing, but the internal Docker registry is reachable via HTTP. It supports the Docker Registry v2 API, allowing OCI image manifest and blob downloads via Python scripts.
4. Tool Interface Specification
The following tools are available to the model for interacting with the sandbox.
Core Execution Tools
| Tool Name | Description | Signature |
|---|---|---|
| container.exec | Executes shell commands (Bash). Returns stdout/stderr. Used for running non-Python languages or system tasks. | exec({ cmd: string[], session_name?: string, workdir?: string, timeout?: number, env?: object, user?: string }) |
| python.exec | Executes Python code for internal reasoning. Output is not shown to the user. | exec(code: string) |
| python_user_visible.exec | Executes Python code where output (plots, tables) is displayed to the user. | exec(code: string) |
| container.feed_chars | Sends keystrokes/input to a running interactive session. | feed_chars({ session_name: string, chars: string }) |
File System & I/O Tools
| Tool Name | Description | Signature |
|---|---|---|
| container.download | Privileged Fetch. Downloads a file from a public URL to a local filepath. Bypasses the network block applied to user code. | download({ url: string, filepath: string }) |
| container.open_image | Displays an image file from the container to the chat UI. | open_image({ path: string }) |
| canmore.create_textdoc | Creates a new text or code file in the Canvas interface. | create_textdoc({ name: string, type: string, content: string }) |
5. Internal Environment Variables
Inspection of the environment reveals the following configuration variables used to wire up the internal proxies.
General Network Policy:
NETWORK=caas_packages_only

Artifact Registry Base:
CAAS_ARTIFACTORY_BASE_URL=packages.applied-caas-gateway1.internal.api.openai.org
CAAS_ARTIFACTORY_READER_USERNAME=reader
CAAS_ARTIFACTORY_READER_PASSWORD=****

Language-Specific Registries:
 * Python: CAAS_ARTIFACTORY_PYPI_REGISTRY
 * NPM: CAAS_ARTIFACTORY_NPM_REGISTRY
 * Go: CAAS_ARTIFACTORY_GO_REGISTRY
 * Maven: CAAS_ARTIFACTORY_MAVEN_REGISTRY
 * Gradle: CAAS_ARTIFACTORY_GRADLE_REGISTRY
 * Cargo (Rust): CAAS_ARTIFACTORY_CARGO_REGISTRY
 * Docker: CAAS_ARTIFACTORY_DOCKER_REGISTRY
 