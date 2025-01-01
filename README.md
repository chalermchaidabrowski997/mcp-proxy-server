# MCP Proxy Server

[简体中文](README_ZH.md)
## ✨ Key Features Highlight

*   **🌐 Web UI Management:** Easily manage all connected MCP servers through an intuitive web interface (optional, requires enabling).
*   **🔧 Granular Tool Control:** Enable or disable individual tools, and override names/descriptions via the Web UI.
*   **🛡️ Flexible Endpoint Authentication:** Secure your HTTP-based endpoints (`/sse`, `/mcp`) with flexible authentication options (`Authorization: Bearer <token>` or `X-API-Key: <key>`).
*   **🔄 Robust Session Handling & Concurrency**:
    *   Improved SSE session handling for client reconnections (relying on server-sent `endpoint` events) and support for concurrent connections.
    *   Streamable HTTP endpoint (`/mcp`) also supports concurrent client interactions.
*   **🚀 Versatile MCP Operations (Server & Proxy):**
    *   **Acts as a Proxy:** Connects to and aggregates multiple backend MCP servers of various types (Stdio, SSE, Streamable HTTP).
    *   **Acts as a Server:** Exposes these aggregated capabilities through its own Streamable HTTP (`/mcp`) and SSE (`/sse`) endpoints. Can also run in a pure Stdio mode.
*   **✨ Real-time Install Output**: Monitor Stdio server installation progress (stdout/stderr) directly in the Web UI.
*   **✨ Web Terminal**: Access a command-line terminal within the Admin UI for direct server interaction (optional, use with caution due to security risks).

---

This server acts as a central hub for Model Context Protocol (MCP) resource servers. It can:

- Connect to and manage multiple backend MCP servers (Stdio, SSE, and Streamable HTTP types).
- Expose their combined capabilities (tools, resources) through a single, unified SSE interface, a Streamable HTTP interface, **or** act as a single Stdio-based MCP server itself.
- Handle routing of requests to the appropriate backend servers.
- Aggregate responses if needed (though primarily acts as a proxy).
- Support multiple simultaneous SSE client connections with optional API key authentication.

## Features

### Resource & Tool Management via Proxy
- Discovers and connects to multiple MCP resource servers defined in `config/mcp_server.json`.
- Aggregates tools and resources from all connected *active* servers.
- Routes tool calls and resource access requests to the correct backend server.
- Maintains consistent URI schemes.

### ✨ Optional Web Admin UI (`ENABLE_ADMIN_UI=true`)
Provides a browser-based interface for managing the proxy server configuration and connected tools. Features include:
- **Server Configuration**: View, add, edit, and delete server entries (`mcp_server.json`). Supports Stdio, SSE, and HTTP server types with relevant options (type, command, args, env, url, apiKey, bearerToken, install config).
- **Tool Configuration**: View all tools discovered from active backend servers. Enable or disable specific tools. Override the display name and description for each tool (`tool_config.json`).
- **Live Reload**: Apply server and tool configuration changes by triggering a configuration reload without needing to restart the entire proxy server process.
- **Stdio Server Installation**: For Stdio servers, you can define installation commands in the configuration. The Admin UI allows you to:
    - Trigger the execution of these installation commands.
    - **Monitor installation progress in real-time** with live stdout and stderr output streamed directly to the UI.
- **Web Terminal**: Access an integrated web-based terminal that provides shell access to the environment where the proxy server is running.
    - **Security Warning**: This feature grants significant access and should be used with extreme caution, especially if the admin interface is exposed.

## Configuration

Configuration is primarily done via environment variables and JSON files located in the `./config` directory.

### 1. Server Connections (`config/mcp_server.json`)
This file defines the backend MCP servers the proxy should connect to.

Example `config/mcp_server.json`:
```json
{
  "mcpServers": {
    "unique-server-key1": {
      "type": "stdio",
      "name": "My Stdio Server",
      "active": true,
      "command": "/path/to/server/executable",
      "args": ["--port", "1234"],
      "env": {
        "API_KEY": "server_specific_key"
      },
      "installDirectory": "/custom_install_path/unique-server-key1",
      "installCommands": [
        "git clone https://github.com/some/repo unique-server-key1",
        "cd unique-server-key1 && npm install && npm run build"
      ]
    },
    "another-sse-server": {
      "type": "sse",
      "name": "My SSE Server",
      "active": true,
      "url": "http://localhost:8080/sse",
      "apiKey": "sse_server_api_key"
    },
    "http-mcp-server": {
      "type": "http",
      "name": "My Streamable HTTP Server",
      "active": true,
      "url": "http://localhost:8081/mcp",
      "bearerToken": "some_secure_token_for_http_server"
    },
    "stdio-default-install": {
        "type": "stdio",
        "name": "Stdio Server with Default Install Path",
        "active": true,
        "command": "my_other_server",
        "installCommands": ["echo 'Installing to default location...'"]
    }
  }
}
```

**Fields:**
-   `mcpServers`: (Required) An object where each key is a unique identifier for a backend server.
-   `name`: (Optional) A user-friendly display name for the server (used in Admin UI).
-   `active`: (Optional, default: `true`) Set to `false` to prevent the proxy from connecting to this server.
-   `type`: (Required) Specifies the transport type. Must be one of `"stdio"`, `"sse"`, or `"http"`.
-   `command`: (Required if `type` is "stdio") The command to execute the server process.
-   `args`: (Optional if `type` is "stdio") An array of string arguments to pass to the command.
-   `env`: (Optional if `type` is "stdio") An object of environment variables (`KEY: "value"`) to set for the server process. These are merged with the proxy server's environment.
-   `url`: (Required if `type` is "sse" or "http") The full URL of the backend server's endpoint (e.g., SSE endpoint for "sse", MCP endpoint for "http").
-   `apiKey`: (Optional if `type` is "sse" or "http") An API key to send in the `X-Api-Key` header when the proxy connects to *this specific backend* server.
-   `bearerToken`: (Optional if `type` is "sse" or "http") A token to send in the `Authorization: Bearer <token>` header when connecting to *this specific backend* server. (If both `apiKey` and `bearerToken` are provided, `bearerToken` generally takes precedence for that specific backend connection).
-   `installDirectory`: (Optional if `type` is "stdio") The absolute path where the server *itself* should be installed (e.g., `/opt/my-server-files`). Used by the Admin UI's installation feature.
    - If provided in `mcp_server.json`, this exact path is used.
    - If omitted, the effective directory depends on the `TOOLS_FOLDER` environment variable (see Environment Variables section).
        - If `TOOLS_FOLDER` is set and not empty, the server will be installed in a subdirectory named after the server key within this folder (e.g., `${TOOLS_FOLDER}/<server_key>`).
        - If `TOOLS_FOLDER` is also empty or not set, it defaults to a `tools` subdirectory within the proxy server's working directory (e.g., `./tools/<server_key>`).
    - Ensure the parent directory of the target installation path (e.g., `TOOLS_FOLDER` or `./tools`) is writable by the user running the proxy server.
-   `installCommands`: (Optional for Stdio type) An array of shell commands executed sequentially by the Admin UI's installation feature if the target server directory (derived from `installDirectory` or defaults) does not exist. Commands are executed from the **parent directory** of the target server installation directory (e.g., if `installDirectory` resolves to `/opt/tools/my-server`, commands run in `/opt/tools/`). **Use with extreme caution due to security risks.**

### 2. Tool Configuration (`config/tool_config.json`)
This file allows overriding properties of tools discovered from backend servers. It is primarily managed via the Admin UI but can be edited manually.

Example `config/tool_config.json`:
```json
{
  "tools": {
    "unique-server-key1__tool-name-from-server": {
      "enabled": true,
      "displayName": "My Custom Tool Name",
      "description": "A more user-friendly description."
    },
    "another-sse-server__another-tool": {
      "enabled": false
    }
  }
}
```
- Keys are in the format `<server_key><separator><original_tool_name>`, where `<separator>` is the value of the `SERVER_TOOLNAME_SEPERATOR` environment variable (defaults to `__`).
- `enabled`: (Optional, default: `true`) Set to `false` to hide this tool from clients connecting to the proxy.
- `displayName`: (Optional) Override the tool's name in client UIs.
- `description`: (Optional) Override the tool's description.

### 3. Environment Variables

-   **`PORT`**: Port for the proxy server's HTTP-based endpoints (`/sse`, `/mcp`, and Admin UI if enabled). Default: `3663`. **Note:** This is only used when running in a mode that starts an HTTP server (e.g., via `npm run dev:sse` or the Docker container). The `npm run dev` script runs in Stdio mode.
    ```bash
    export PORT=8080
    ```
-   **`ALLOWED_KEYS`**: (Optional) Comma-separated list of API keys to secure the proxy's HTTP-based endpoints (`/sse`, `/mcp`). If neither `ALLOWED_KEYS` nor `ALLOWED_TOKENS` are set, authentication is disabled for these endpoints. Clients must provide a key via `X-Api-Key` header or `?key=` query parameter.
    ```bash
    export ALLOWED_KEYS="client_key1,client_key2"
    ```
-   **`ALLOWED_TOKENS`**: (Optional) Comma-separated list of Bearer Tokens to secure the proxy's HTTP-based endpoints (`/sse`, `/mcp`). If neither `ALLOWED_KEYS` nor `ALLOWED_TOKENS` are set, authentication is disabled. Clients must provide a token via the `Authorization: Bearer <token>` header. If both `ALLOWED_KEYS` and `ALLOWED_TOKENS` are configured, Bearer Token authentication will be attempted first.
    ```bash
    export MCP_PROXY_SSE_ALLOWED_TOKENS="your_bearer_token_1,your_bearer_token_2"
    ```
-   **`ENABLE_ADMIN_UI`**: (Optional) Set to `true` to enable the Web Admin UI (only applicable in SSE mode). Default: `false`.
    ```bash
    export ENABLE_ADMIN_UI=true
    ```
-   **`ADMIN_USERNAME`**: (Required if Admin UI enabled) Username for Admin UI login. Default: `admin`.
-   **`ADMIN_PASSWORD`**: (Required if Admin UI enabled) Password for Admin UI login. Default: `password` (**Change this!**).
    ```bash
    export ADMIN_USERNAME=myadmin
    export ADMIN_PASSWORD=aVerySecurePassword123!
    ```
-   **`SESSION_SECRET`**: (Optional, recommended if Admin UI enabled) Secret used to sign session cookies. If not set, a default, less secure secret is used, and a warning is issued. A secure secret is automatically generated and saved to `config/.session_secret` on first run if not provided via environment variable.
    ```bash
    # Recommended: Generate a strong secret (e.g., openssl rand -hex 32)
    export SESSION_SECRET='your_very_strong_random_secret_here'
    ```
-   **`TOOLS_FOLDER`**: (Optional) Specifies the base directory for Stdio server installations initiated via the Admin UI, used when `installDirectory` is not explicitly set in `mcp_server.json` for a specific server.
    - If set (e.g., `/custom/tools_path`), installations for servers without a specific `installDirectory` will target a subdirectory named after the server key within this folder (e.g., `${TOOLS_FOLDER}/<server_key>`).
    - If `TOOLS_FOLDER` is not set or is empty, such installations will default to a `tools` subdirectory within the proxy server's working directory (e.g., `./tools/<server_key>`).
    - The Dockerfile sets this to `/tools` by default.
    ```bash
    export TOOLS_FOLDER=/srv/mcp_tools
    ```

-   **`SERVER_TOOLNAME_SEPERATOR`**: (Optional) Defines the separator used to combine the server name and tool name when generating the unique key for tools (e.g., `server-key__tool-name`). This key is used internally and in the `tool_config.json` file.
    -   Default: `__`.
    -   Must be at least 2 characters long and contain only letters (a-z, A-Z), numbers (0-9), hyphens (`-`), and underscores (`_`).
    -   If the provided value is invalid, the default (`__`) will be used, and a warning will be logged.
    ```bash
    export SERVER_TOOLNAME_SEPERATOR="___" # Example: using triple underscore
    ```

-   **`LOGGING`**: (Optional) Controls the minimum log level output by the server.
    -   Possible values (case-insensitive): `error`, `warn`, `info`, `debug`.
    -   Logs at the specified level and all levels above it will be shown.
    -   Default: `info`.
    ```bash
    export LOGGING="debug"
    ```

-   **`RETRY_SSE_TOOL_CALL`**: (Optional) Controls whether to enable retries for SSE tool calls. Set to `"true"` to enable, `"false"` to disable. Default: `true`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export RETRY_SSE_TOOL_CALL="true"
    ```
-   **`SSE_TOOL_CALL_MAX_RETRIES`**: (Optional) Maximum number of retry attempts for SSE tool calls (after the initial failure). Default: `2`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export SSE_TOOL_CALL_MAX_RETRIES="2"
    ```
-   **`SSE_TOOL_CALL_RETRY_DELAY_BASE_MS`**: (Optional) Base delay in milliseconds for SSE tool call retries, used in exponential backoff. Default: `300`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export SSE_TOOL_CALL_RETRY_DELAY_BASE_MS="300"
    ```
-   **`RETRY_HTTP_TOOL_CALL`**: (Optional) Controls whether to retry on HTTP tool call connection errors. Set to `"true"` to enable, `"false"` to disable. Default: `true`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export RETRY_HTTP_TOOL_CALL="true"
    ```
-   **`HTTP_TOOL_CALL_MAX_RETRIES`**: (Optional) Maximum number of retry attempts for HTTP tool calls (after the initial failure). Default: `2`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export HTTP_TOOL_CALL_MAX_RETRIES="3"
    ```
-   **`HTTP_TOOL_CALL_RETRY_DELAY_BASE_MS`**: (Optional) Base delay in milliseconds for HTTP tool call retries, used in exponential backoff. Default: `300`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export HTTP_TOOL_CALL_RETRY_DELAY_BASE_MS="500"
    ```
-   **`RETRY_STDIO_TOOL_CALL`**: (Optional) Controls whether to retry on Stdio tool call connection errors (attempts to restart the process). Set to `"true"` to enable, `"false"` to disable. Default: `true`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export RETRY_STDIO_TOOL_CALL="true"
    ```
-   **`STDIO_TOOL_CALL_MAX_RETRIES`**: (Optional) Maximum number of retry attempts for Stdio tool calls (after the initial failure). Default: `2`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export STDIO_TOOL_CALL_MAX_RETRIES="5"
    ```
-   **`STDIO_TOOL_CALL_RETRY_DELAY_BASE_MS`**: (Optional) Base delay in milliseconds for Stdio tool call retries, used in exponential backoff. Default: `300`. See the "Enhanced Reliability Features" section for details.
    ```bash
    export STDIO_TOOL_CALL_RETRY_DELAY_BASE_MS="1000"
    ```

## Enhanced Reliability Features

The MCP Proxy Server includes features to improve its resilience and the reliability of interactions with backend MCP services, ensuring smoother operations and more consistent tool execution.

### 1. Error Propagation
The proxy server ensures that errors originating from backend MCP services are consistently propagated to the requesting client. These errors are formatted as standard JSON-RPC error responses, making it easier for clients to handle them uniformly.

### 2. SSE Tool Call Retry
When a `tools/call` operation is made to an SSE-based backend server, and the underlying connection is lost or experiences an error (including timeouts), the proxy server implements a retry mechanism.

**Retry Mechanism:**
If an initial SSE tool call fails due to a connection error or timeout, the proxy will attempt to re-establish the connection to the SSE backend. If reconnection is successful, it will then retry the original `tools/call` request using an exponential backoff strategy, similar to HTTP and Stdio retries. This means the delay before each subsequent retry attempt increases exponentially, with a small amount of jitter (randomness) added.

**Configuration:**
These settings are primarily controlled by environment variables. Values in `config/mcp_server.json` under the `proxy` object for these specific keys will be overridden by environment variables if set.

-   **`RETRY_SSE_TOOL_CALL`** (environment variable):
    -   Set to `"true"` to enable retries for SSE tool calls.
    -   Set to `"false"` to disable this feature.
    -   **Default Behavior:** `true` (if the environment variable is not set, is empty, or is an invalid value).

-   **`SSE_TOOL_CALL_MAX_RETRIES`** (environment variable):
    -   Specifies the maximum number of retry attempts *after* the initial failed attempt. For example, if set to `"2"`, there will be one initial attempt and up to two retry attempts, totaling a maximum of three attempts.
    -   **Default Behavior:** `2` (if the environment variable is not set, is empty, or is not a valid integer).

-   **`SSE_TOOL_CALL_RETRY_DELAY_BASE_MS`** (environment variable):
    -   The base delay in milliseconds used in the exponential backoff calculation. The delay before the *n*-th retry (0-indexed) is roughly `SSE_TOOL_CALL_RETRY_DELAY_BASE_MS * (2^n) + jitter`.
    -   **Default Behavior:** `300` (milliseconds) (if the environment variable is not set, is empty, or is not a valid integer).

**Example (Environment Variables):**
```bash
export RETRY_SSE_TOOL_CALL="true"
export SSE_TOOL_CALL_MAX_RETRIES="3"
export SSE_TOOL_CALL_RETRY_DELAY_BASE_MS="500"
```

### 3. HTTP Request Retry for Tool Calls
For `tools/call` operations directed to HTTP-based backend servers, the proxy implements a retry mechanism for connection errors (e.g., "failed to fetch", network timeouts).

**Retry Mechanism:**
If an initial HTTP request fails due to a connection error, the proxy will retry the request using an exponential backoff strategy. This means the delay before each subsequent retry attempt increases exponentially, with a small amount of jitter (randomness) added to prevent thundering herd scenarios.

**Configuration:**
These settings are primarily controlled by environment variables.

-   **`RETRY_HTTP_TOOL_CALL`** (environment variable):
    -   Set to `"true"` to enable retries for HTTP tool calls.
    -   Set to `"false"` to disable this feature.
    -   **Default Behavior:** `true` (if the environment variable is not set, is empty, or is an invalid value).

-   **`HTTP_TOOL_CALL_MAX_RETRIES`** (environment variable):
    -   Specifies the maximum number of retry attempts *after* the initial failed attempt. For example, if set to `"2"`，there will be one initial attempt and up to two retry attempts, totaling a maximum of three attempts.
    -   **Default Behavior:** `2` (if the environment variable is not set, is empty, or is not a valid integer).

-   **`HTTP_TOOL_CALL_RETRY_DELAY_BASE_MS`** (environment variable):
    -   The base delay in milliseconds used in the exponential backoff calculation. The delay before the *n*-th retry (0-indexed) is roughly `HTTP_TOOL_CALL_RETRY_DELAY_BASE_MS * (2^n) + jitter`.
    -   **Default Behavior:** `300` (milliseconds) (if the environment variable is not set, is empty, or is not a valid integer).

### 4. Stdio Connection Retry for Tool Calls
For `tools/call` operations directed to Stdio-based backend servers, the proxy implements a retry mechanism for connection errors (e.g., process crash or unresponsiveness).

**Retry Mecha