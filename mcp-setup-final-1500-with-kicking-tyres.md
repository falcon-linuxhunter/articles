
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

- **Docker Desktop** installed (for Windows/macOS)
- **MCP-compatible client** (Claude Desktop, Cursor, Windsurf)
- **Brave Search API Key** (optional for internet search)
- **Postgres Database credentials** (optional for database integration)

On Windows, also ensure WSL2 is installed.

---


## Platform Setup

### 🪟 Windows (WSL2 + Docker Desktop)

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

3. Enable WSL2 integration for Docker Desktop. 

   ![image info](./images/enable_wsl2_docker.png)

4. Optionally, enable Ubuntu in Docker Desktop.


   ![image info](./images/optinal_enable.png)

5. Test Docker using PowerShell:

   ```bash
   docker run hello-world
   ```
   ![image info](./images/test_docker.png)



### 🍎 macOS

1. Download and install Docker Desktop for Mac.
2. Verify Docker installation:
   ```bash
   docker run hello-world
   ```
3. Plan directories like `~/Projects` or `~/Desktop/MyCode` for safe mounting.

> **Linux Note**: Follow the macOS instructions if you're setting up on Linux.

---

## Setting Up MCP Tools

We will configure three core MCP tools: Filesystem, Brave Search, and PostgreSQL.

Each tool runs as an isolated Docker container that your AI client can launch automatically.

### 1. Filesystem Tool

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
