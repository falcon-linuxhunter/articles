
# Setting Up Model Context Protocol (MCP) Servers Locally with Docker

Model Context Protocol (MCP) is a secure framework introduced by Anthropic that enables AI assistants to safely interact with external tools and data sources. 
By setting up MCP servers locally, you can empower coding assistants like Claude Desktop, Windsurf (Cascade), or Cursor to access your local filesystem, perform live web searches, or query databases securely.

This guide walks you through installing and configuring essential MCP Servers using Docker — with full support for both Windows and macOS. 

---

## Introduction

AI coding assistants are powerful, but connecting them to real-world data amplifies their capabilities. 
Imagine your AI being able to:

- Open and edit local files
- Perform up-to-date internet searches
- Run SQL queries on your databases

By setting up local MCP servers, you keep control over what resources are exposed while giving your AI superpowers.

Let's dive in! 🚀

---

## Prerequisites

Before proceeding, make sure you have:

- **WSL2 & Ubuntu** (Windows Only)
- **Docker Desktop** installed (for Windows/macOS)
- **MCP-compatible client** (Claude Desktop, Cursor, Windsurf)
- **Brave Search API Key** (optional for internet search)
- **Postgres Database credentials** (optional for database integration)

---


## Platform Setup

### 🪟 Windows (WSL2 + Ubuntu + Docker Desktop)

From the Start menu in Windows 10/11, search for **"Turn Windows features on or off"** and select it.

![image info](./images/windows_features.png)

You need to enable two options here:
1. Virtual Machine Platform
2. Windows Subsystem for Linux

![image info](./images/subsystem.png)

Apply the changes, and the system will prompt you to restart!

> **Note:** You must enable virtualization from the BIOS.  
> To check if virtualization is enabled, open **Task Manager** and look under the **Performance** tab!


![image info](./images/virtualization.png)

1. Open **PowerShell** as **Administrator** and run the following commands:
   ```bash
   wsl --version
   ```
    ![image info](./images/wsl_version.png)

    ```bash
   wsl --install
   ```
    ![image info](./images/install_ubuntu.png)
  
    Now that Ubuntu is installed, you can open it directly from the **Start Menu**. Provide a **username** and **password**.

    ![image info](./images/open_ubuntu.png)




2. [Download](https://www.docker.com/products/docker-desktop/) and install Docker Desktop for Windows.

3. Start Docker Desktop from the Start Menu.

4. Enable WSL2 integration for Docker Desktop. 

   ![image info](./images/enable_wsl2_docker.png)

5. Enable Ubuntu in Docker Desktop.


   ![image info](./images/optinal_enable.png)
  
6. Apply & restart Docker Desktop.

7. Test Docker Desktop using Ubuntu terminal in Windows :

   ```bash
   docker run hello-world
   ```
   ![image info](./images/test_docker.png)


### 🍎 macOS

1. [Download](https://www.docker.com/products/docker-desktop/) and install Docker Desktop for Mac.

2. Start Docker Desktop from the Applications menu.

3. Verify Docker installation using the terminal:

   ```bash
   docker run hello-world
   ```


> **Linux Note**: Follow the macOS instructions if you're setting up on Linux.

---

## Setting Up MCP Servers

We will configure multiple MCP servers such as Filesystem, Brave Search, PostgreSQL, and others.

Each server runs as an isolated Docker container that your AI client can launch automatically.

Below is a list of the MCP servers we will set up:

| MCP Server           | Description                                  | Notes / Environment Variables |
|----------------------|----------------------------------------------|--------------------------------|
| Git                  | Git repository access and operations        | None |
| Time                 | Time and timezone utilities                 | None |
| Sqlite               | SQLite database interactions                | None |
| Fetch                | Web content fetching and parsing            | None |
| Sequential Thinking  | Dynamic thought sequence problem-solving    | None |
| Puppeteer            | Browser automation and web scraping         | `DOCKER_CONTAINER=true` |
| PostgreSQL           | PostgreSQL database access                  | Connection string: `postgresql://host.docker.internal:5432/mydb` |
| Memory               | Persistent knowledge graph memory           | None |
| Google Maps          | Location services and directions            | `GOOGLE_MAPS_API_KEY=YOUR-KEY-HERE` |
| Filesystem           | Secure local file system operations         | Folder bind mount required: `/Users/YOUR_FOLDER/bot-env -> /projects/bot-env` |
| Brave Search         | Web and local search via Brave API           | `BRAVE_API_KEY=YOUR-KEY-HERE` |

---

## Build MCP Server Images with Docker

Follow these steps based on your operating system:

### 🐧 Windows (WSL2 Ubuntu) and 🖥️ macOS Users

1. **Clone the Repository**  
   Clone the MCP servers repository to your local machine:
   ```bash
   git clone https://github.com/modelcontextprotocol/servers.git
   ```

2. **Navigate to the Repository Directory**: Change directory to the cloned repository:

    ```bash
    cd servers
    ```

3. **Create a new file**: Create a **build_all.sh** file in the root directory of the project and paste the following content:

    ```bash
    #!/bin/bash
    set -euo pipefail

    # Determine the directory of this script so that relative paths are correct
    SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
    DOCKER_SRC_DIR="$SCRIPT_DIR/src"

    # Loop over each subdirectory inside the src folder
    for folder in "$DOCKER_SRC_DIR"/*; do
        if [ -d "$folder" ]; then
            folder_name=$(basename "$folder")
            
            # Skip 'redis' folder
            if [ "$folder_name" == "redis" ]; then
                echo "Skipping '$folder_name': Redis does not require build"
                continue
            fi

            dockerfile="$folder/Dockerfile"
            if [ -f "$dockerfile" ]; then
                # If Dockerfile references uv.lock, ensure the file exists in the project folder
                if grep -q "uv\.lock" "$dockerfile"; then
                    if [ ! -f "$folder/uv.lock" ]; then
                        echo "Skipping '$folder_name': Dockerfile references uv.lock but file not found in $folder"
                        continue
                    fi
                fi

                # Determine build context:
                # If the Dockerfile contains "COPY src/<folder_name>", it expects to be run from the repository root.
                if grep -E -q "COPY[[:space:]]+src/${folder_name}\b" "$dockerfile"; then
                    context="$SCRIPT_DIR"
                    echo "Building Docker image for '$folder_name' using $dockerfile with repository root as context"
                else
                    context="$folder"
                    echo "Building Docker image for '$folder_name' using $dockerfile with project folder as context"
                fi

                docker build -t "$folder_name" -f "$dockerfile" "$context"
            else
                echo "Skipping '$folder_name': Dockerfile not found"
            fi
        fi
    done


    ```

4. **Make Script Executable**: Make shell script executable:

    ```bash
    chmod +x build_all.sh
    ``` 
5. **Run the Script with `sudo`**  

   The `build_all.sh` script needs to be run with root access (`sudo`) because it builds Docker images, which require elevated permissions.  
   Ensure your user account has the necessary privileges.

   When you run the script, it will:
   - Automatically detect each MCP server directory under the `src/` folder.
   - Check if a valid `Dockerfile` exists for that server.
   - Verify if any additional dependencies (like `uv.lock`) are required.
   - Select the correct Docker build context (either the project root or the individual service folder).
   - Build a Docker image for each MCP server and tag it with the server name (e.g., `git`, `fetch`, `filesystem`, `brave-search`, etc.).

   > ⚡ **Note:** This process will sequentially build multiple images. The time it takes depends on your machine's resources and internet speed (for downloading base images).

   To run the script:
   ```bash
   sudo ./build_all.sh
   ```

   While the script is running, you may encounter multiple errors. Specifically, if you encounter an error like:

   ```bash
   ERROR [internal] load metadata for ghcr.io/astral-sh/uv:python3.12-bookworm-slim
   ```
    it usually means there was a temporary network or connectivity issue while pulling base images.

      ✅ Solution: Simply switch to a different internet connection (e.g., a mobile hotspot, another Wi-Fi network) and rerun the script. You may need to run the script multiple times, as it builds all images sequentially. Since this project is open-source and the remote repositories are updated frequently, changes can occasionally break the build process. For advanced use cases, you might need to adjust the script to handle specific services. For example, the Redis MCP server's Dockerfile is currently broken, so we have modified the script to skip building it.

6. **Verify MCP Server Docker Images**  
   Once the script finishes, you can verify that the Model Context Protocol (MCP) server Docker images were built successfully by running:

    ```bash
    sudo docker images
    ```
      ![image info](./images/docker_images.png)

    










































## 1. Filesystem Tool

Allows your AI to securely read/write local files.

Docker Pull:
```bash
docker pull mcp/filesystem
```

Example Configuration:
```json
"filesystem": {
  "command": "docker",
  "args": [
    "run", "-i", "--rm",
    "--mount", "type=bind,src=/path/to/desktop,dst=/projects/Desktop",
    "mcp/filesystem",
    "/projects"
  ]
}
```

> Replace `/path/to/desktop` with your actual local path.

---

### 2. Brave Search Tool

Enables live web searches.

Docker Pull:
```bash
docker pull mcp/brave-search
```

Example Configuration:
```json
"brave-search": {
  "command": "docker",
  "args": [
    "run", "-i", "--rm",
    "-e", "BRAVE_API_KEY",
    "mcp/brave-search"
  ],
  "env": {
    "BRAVE_API_KEY": "YOUR_API_KEY_HERE"
  }
}
```

---

### 3. PostgreSQL Tool

Securely query local or remote databases.

Docker Pull:
```bash
docker pull mcp/postgres
```

Example Configuration:
```json
"postgres-db": {
  "command": "docker",
  "args": [
    "run", "-i", "--rm",
    "-e", "DATABASE_URL",
    "mcp/postgres"
  ],
  "env": {
    "DATABASE_URL": "postgresql://user:pass@host.docker.internal:5432/dbname"
  }
}
```

Use `host.docker.internal` on Windows/macOS for DB networking.

---

## Combined Sample `mcpServers.json`

```json
{
  "mcpServers": {
    "filesystem": { ... },
    "brave-search": { ... },
    "postgres-db": { ... }
  }
}
```

Save it according to your client:

| Client         | Location |
|----------------|----------|
| Claude Desktop (Windows) | `%APPDATA%/Claude/claude_desktop_config.json` |
| Claude Desktop (macOS) | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Cursor | `.cursor/mcp.json` per project |

Restart the client after saving.

---

## Kicking the Tyres: Testing Your Setup 🛞

Now it's time to **kick the tyres** and make sure everything is running smoothly!

### Test the Filesystem Server
Ask your AI:
> "List the files inside the Desktop folder."

If configured correctly, your assistant should list real files.

### Test Brave Search
Prompt:
> "Search the web for the latest Python 3.12 updates."

It should return search results from Brave.

### Test PostgreSQL Access
Ask:
> "List the tables in the `mydatabase` Postgres database."

You should receive a proper table listing.

If something doesn't work — don't worry, kicking the tyres helps you find and fix early!

---

## Common Troubleshooting

| Problem | Solution |
|---------|----------|
| Container doesn't start | Check Docker Desktop settings, WSL2 integration |
| Filesystem mount errors | Validate your mount path in config |
| Brave Search failure | Confirm your API Key is correct |
| Database connection error | Ensure your database is reachable and URL is correct |

Use Docker commands to diagnose:
```bash
docker ps -a
docker logs <container_id>
```

And prune unused containers:
```bash
docker system prune
```

---

## Security Best Practices 🔒

- **Limit mounts**: Only expose needed folders.
- **Use read-only mounts** where possible.
- **Secure secrets**: Pass API keys using environment variables, never hardcode.
- **Restrict database permissions**: Use a read-only Postgres user.
- **Update Docker images** regularly to patch vulnerabilities.
- **Watch resource usage**: Containers spawn on-demand; monitor with `docker stats` if needed.
- **Network isolation**: Prefer `STDIO` over opening ports unless absolutely necessary.

---

## Conclusion and What's Next? 🚀

You’ve now successfully:

- Built local MCP tool servers with Docker
- Wired them into your AI assistant securely
- Tested real-world file access, web search, and database querying

This is just the beginning — the MCP ecosystem is growing rapidly.  
You can explore additional servers like:

- GitHub integration (code search)
- Slack messaging
- Local Docker control

Each new tool you add makes your AI smarter and your workflow faster.

Stay updated with new MCP server releases and happy coding! 🎉
