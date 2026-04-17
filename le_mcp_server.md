# LE — MCP: Model Context Protocol — Tool-Server bauen & verteilen
> Lernblock 4: AI-Fähigkeiten zentralisieren · 1 Stunde

---

## 1. Was ist MCP? (15 min)

MCP (Model Context Protocol) ist ein **offener Standard** der Anthropic für die Anbindung von externen Tools und Datenquellen an Claude. Denkweise: wie USB für AI-Tools.

```
OHNE MCP (bisher):
  Jede Anwendung definiert Tools selbst (JSON-Schema, LE 06)
  → Keine Standardisierung
  → Nicht wiederverwendbar
  → Muss für jedes Projekt neu gebaut werden

MIT MCP:
  MCP Server = standardisiertes "Tool-Plugin"
  ┌──────────────────────────────────────────────────────────┐
  │  Claude Code / Claude Desktop / Deine App               │
  │          │ MCP-Protokoll (JSON-RPC über stdio/HTTP)      │
  │          ▼                                               │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
  │  │  JIRA MCP    │  │  Git MCP     │  │  DB MCP      │  │
  │  │  Server      │  │  Server      │  │  Server      │  │
  │  └──────────────┘  └──────────────┘  └──────────────┘  │
  └──────────────────────────────────────────────────────────┘
  
  → Einmal gebaut = überall nutzbar (Claude Code, Desktop, API)
  → Community baut MCP-Server, du nutzt sie
  → Dein Team baut firmeninterne MCP-Server, alle nutzen sie
```

### MCP bietet drei Bausteine

```
┌─────────────────────────────────────────────────────────────┐
│  TOOLS      → Claude kann Funktionen aufrufen               │
│               (Daten lesen, schreiben, API-Calls)           │
│               Beispiel: jira.getTicket("PROJ-123")          │
│                                                             │
│  RESOURCES  → Claude kann Dateien/Dokumente lesen           │
│               Beispiel: confluence://design-doc/auth        │
│               → Ähnlich wie RAG, aber strukturierter        │
│                                                             │
│  PROMPTS    → Vordefinierte Prompt-Templates                │
│               Beispiel: /code-review-template               │
│               → Ähnlich wie Slash Commands, aber im Server  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Eingebaute MCP-Server nutzen (10 min)

Claude Code hat bereits einige MCP-Server verfügbar. Konfiguration in `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_..."
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres",
               "postgresql://user:pass@localhost/mydb"]
    }
  }
}
```

### Bekannte öffentliche MCP-Server

```
@modelcontextprotocol/server-filesystem   → Dateisystem-Zugriff
@modelcontextprotocol/server-github       → GitHub API (Issues, PRs, Code)
@modelcontextprotocol/server-postgres     → PostgreSQL direkt befragen
@modelcontextprotocol/server-slack        → Slack Nachrichten lesen/schreiben
@modelcontextprotocol/server-brave-search → Web-Suche via Brave
@modelcontextprotocol/server-puppeteer    → Browser-Automation

→ Vollständige Liste: https://github.com/modelcontextprotocol/servers
```

---

## 3. Eigenen MCP-Server bauen (Python) (20 min)

### JIRA-MCP-Server für dein Team

```python
# jira_mcp_server.py
# pip install mcp jira

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types
from jira import JIRA
import os
import json

# JIRA-Verbindung
jira = JIRA(
    server=os.environ["JIRA_URL"],
    basic_auth=(os.environ["JIRA_USER"], os.environ["JIRA_TOKEN"])
)

app = Server("jira-mcp")

# ── TOOLS definieren ─────────────────────────────────────

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_ticket",
            description="Liest ein JIRA-Ticket anhand der ID. Gibt Titel, Status, Beschreibung und Kommentare zurück.",
            inputSchema={
                "type": "object",
                "properties": {
                    "ticket_id": {
                        "type": "string",
                        "description": "JIRA-Ticket-ID, z.B. PROJ-123"
                    }
                },
                "required": ["ticket_id"]
            }
        ),
        types.Tool(
            name="search_tickets",
            description="Sucht JIRA-Tickets per JQL-Query. Gibt Liste von Ticket-IDs und Titeln zurück.",
            inputSchema={
                "type": "object",
                "properties": {
                    "jql": {
                        "type": "string",
                        "description": "JQL-Suchquery, z.B. 'project = PROJ AND status = Open'"
                    },
                    "max_results": {
                        "type": "integer",
                        "description": "Maximale Anzahl Ergebnisse (default: 10)",
                        "default": 10
                    }
                },
                "required": ["jql"]
            }
        ),
        types.Tool(
            name="create_ticket",
            description="Erstellt ein neues JIRA-Ticket.",
            inputSchema={
                "type": "object",
                "properties": {
                    "project": {"type": "string", "description": "Projekt-Key, z.B. PROJ"},
                    "summary": {"type": "string", "description": "Ticket-Titel"},
                    "description": {"type": "string"},
                    "issue_type": {
                        "type": "string",
                        "enum": ["Bug", "Story", "Task", "Subtask"],
                        "default": "Task"
                    }
                },
                "required": ["project", "summary"]
            }
        )
    ]

# ── TOOL-AUSFÜHRUNG ──────────────────────────────────────

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    
    if name == "get_ticket":
        issue = jira.issue(arguments["ticket_id"])
        result = {
            "id": issue.key,
            "summary": issue.fields.summary,
            "status": issue.fields.status.name,
            "description": issue.fields.description or "Keine Beschreibung",
            "assignee": str(issue.fields.assignee) if issue.fields.assignee else "Unassigned",
            "priority": issue.fields.priority.name if issue.fields.priority else "None"
        }
        return [types.TextContent(type="text", text=json.dumps(result, ensure_ascii=False))]
    
    if name == "search_tickets":
        issues = jira.search_issues(
            arguments["jql"],
            maxResults=arguments.get("max_results", 10)
        )
        results = [{"id": i.key, "summary": i.fields.summary, "status": i.fields.status.name}
                   for i in issues]
        return [types.TextContent(type="text", text=json.dumps(results, ensure_ascii=False))]
    
    if name == "create_ticket":
        issue = jira.create_issue(
            project=arguments["project"],
            summary=arguments["summary"],
            description=arguments.get("description", ""),
            issuetype={"name": arguments.get("issue_type", "Task")}
        )
        return [types.TextContent(type="text", text=f"Ticket erstellt: {issue.key}")]
    
    return [types.TextContent(type="text", text=f"Unbekanntes Tool: {name}")]

# ── RESOURCES (optional): Aktuelle Sprints als lesbare Ressource ──

@app.list_resources()
async def list_resources() -> list[types.Resource]:
    return [
        types.Resource(
            uri="jira://active-sprint",
            name="Aktueller Sprint",
            description="Alle Tickets im aktuellen Sprint",
            mimeType="application/json"
        )
    ]

@app.read_resource()
async def read_resource(uri: str) -> str:
    if uri == "jira://active-sprint":
        issues = jira.search_issues(
            "sprint in openSprints() AND assignee = currentUser()",
            maxResults=20
        )
        tickets = [{"id": i.key, "summary": i.fields.summary, "status": i.fields.status.name}
                   for i in issues]
        return json.dumps(tickets, ensure_ascii=False, indent=2)
    raise ValueError(f"Unbekannte Resource: {uri}")

# ── STARTEN ──────────────────────────────────────────────

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream, app.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### In Claude Code einbinden

```json
// .claude/settings.json (Projekt-Level → geteilt via Git!)
{
  "mcpServers": {
    "jira": {
      "command": "python3",
      "args": ["/shared/tools/jira_mcp_server.py"],
      "env": {
        "JIRA_URL": "https://yourcompany.atlassian.net",
        "JIRA_USER": "user@company.com",
        "JIRA_TOKEN": "${JIRA_API_TOKEN}"   // ← aus Umgebungsvariable!
      }
    }
  }
}
```

---

## 4. MCP in Java (Spring Boot MCP Server) (10 min)

```java
// pom.xml:
// <dependency>
//   <groupId>io.modelcontextprotocol.sdk</groupId>
//   <artifactId>mcp-spring-webmvc</artifactId>
// </dependency>

@SpringBootApplication
public class JavaMcpServer {
    
    @Bean
    public ToolCallbackProvider tools(JiraService jiraService) {
        // Spring AI MCP Tool-Registration
        return MethodToolCallbackProvider.builder()
            .toolObjects(jiraService)
            .build();
    }
    
    public static void main(String[] args) {
        SpringApplication.run(JavaMcpServer.class, args);
    }
}

@Service
public class JiraService {
    
    @Tool(description = "Liest ein JIRA-Ticket anhand der ID")
    public String getTicket(
        @ToolParam(description = "Ticket-ID z.B. PROJ-123") String ticketId
    ) {
        // JIRA REST API aufrufen
        return jiraRestClient.getIssue(ticketId).toJson();
    }
}
```

```json
// Claude Code Konfiguration für Java MCP Server:
{
  "mcpServers": {
    "jira-java": {
      "command": "java",
      "args": ["-jar", "/tools/jira-mcp-server.jar"],
      "env": {
        "JIRA_TOKEN": "${JIRA_API_TOKEN}"
      }
    }
  }
}
```

---

## 5. Distribution & Wartung (5 min)

```
MCP-SERVER VERTEILEN:

Option A: Git-Repository (intern)
  → MCP-Server Code in zentralem Repo
  → Settings.json in Projekt-Repos per Copy oder Submodul
  → Updates: git pull + Neustart

Option B: Docker-Container
  docker run -e JIRA_TOKEN=... company/jira-mcp-server
  → Isolation, versioniert, deployment-unabhängig
  → Konfiguration in settings.json: "command": "docker", "args": ["run", ...]

Option C: npm-Package (für Node.js MCP-Server)
  npm install -g @company/jira-mcp-server
  → Wie öffentliche MCP-Server
  → Zentrale Verwaltung via npm update

Sicherheit:
  → MCP-Server laufen mit eigenem Prozess und eigenen Rechten
  → Secrets immer per Umgebungsvariable, nie hardcodiert
  → Prinzip: MCP-Server nur die Rechte geben die er braucht
```

---

## Zusammenfassung

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: MCP                                      │
│                                                         │
│  MCP = USB-Standard für AI-Tools                        │
│  → Einmal bauen, überall nutzen                         │
│                                                         │
│  3 Bausteine:                                           │
│    Tools     → Funktionen die Claude aufrufen kann      │
│    Resources → Dokumente die Claude lesen kann          │
│    Prompts   → Templates die Claude nutzen kann         │
│                                                         │
│  Aufbau:                                                │
│    Python: mcp SDK + @app.list_tools() + @app.call_tool()│
│    Java:   Spring Boot + @Tool Annotation               │
│                                                         │
│  Distribution: Git / Docker / npm                       │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [Custom Slash Commands](le_skills_slash_commands.md)*
*Weiter: [Zentrale AI-Infrastruktur für Teams](le_team_ai_infrastruktur.md)*
