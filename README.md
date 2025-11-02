# Official Specification for `go-agent` (Lattice-Agent)

`Lattice-Agent` is a framework for building production-ready AI agents in Go. It provides clean abstractions for large language models (LLMs), tool calling, a sophisticated retrieval-augmented memory system, and multi-agent coordination. The project's philosophy emphasizes performance, scalability, and simplicity with minimal dependencies.

---

### **1. Core Concepts & Architecture**

The framework is designed to be modular, allowing developers to compose agents from reusable components. This is facilitated by an **Agent Development Kit (ADK)**.

#### **1.1. Agent Development Kit (ADK)**

The ADK provides a declarative way to build agents by combining modules for different functionalities like LLM integration, memory, and tooling.

**Example from `lattice-code/src/agent.go`:**
```go
builder, err := adk.New(
    ctx,
    adk.WithDefaultSystemPrompt(VibeSystemPrompt),
    adk.WithModules(
        modules.InQdrantMemory(100000, *qdrantURL, *qdrantCollection, memory.AutoEmbedder(), &memOpts),
        adkmodules.NewModelModule("gemini", func(_ context.Context) (models.Agent, error) {
            return models.NewGeminiLLM(ctx, "gemini-2.5-pro", "Universal code generator")
        }),
        adkmodules.NewToolModule("essentials",
            adkmodules.StaticToolProvider([]agent.Tool{&tools.EchoTool{}}, nil),
        ),
    ),
)
// ...
return builder.BuildAgent(ctx)
```

#### **1.2. Multi-Agent Coordination (Swarm & Spaces)**

Lattice-Agent has first-class support for multi-agent systems, referred to as "swarms".

*   **Participants**: Individual agents within a swarm.
*   **Shared Spaces**: Agents can join shared memory spaces (e.g., `team:design`, `product:beta`) to collaborate and share context in real-time. This allows for building complex workflows with specialist agents.
*   **Orchestration**: A central swarm orchestrator manages the participants and their interactions, as demonstrated in `agent/cmd/team/main.go`.

---

### **2. Memory Engine**

A key feature of Lattice-Agent is its advanced, RAG-powered memory system designed for complex reasoning and context retrieval.

#### **2.1. Hybrid Memory Model: Graph + Multi-Vector**

The memory architecture combines a knowledge graph with multi-vector embeddings to capture both structured relationships and semantic nuance.

*   **Graph-Aware Memory**: Models entities (people, projects) as nodes and their relationships as edges. This allows for precise, structured queries (e.g., "Who manages Project X?").
*   **Multi-Vector Storage**: Instead of a single vector, each entity can have multiple embeddings representing different facets (e.g., role, skills, communication style). This enables more targeted semantic searches.
*   **Retrieval Process**: Retrieval is a two-step process:
    1.  **Semantic Search**: Find relevant nodes using vector search across one or more vector spaces.
    2.  **Graph Traversal**: Explore the connections of the retrieved nodes to gather more context and construct a comprehensive answer.

#### **2.2. Features**

*   **Importance Scoring**: Automatically weights memories by relevance.
*   **MMR (Maximal Marginal Relevance) Retrieval**: Ensures diversity in retrieved results.
*   **Auto-Pruning**: Removes stale or low-value memories to manage context.
*   **Pluggable Backends**: The memory engine supports multiple storage backends:
    *   In-memory (for development)
    *   PostgreSQL with `pgvector`
    *   Qdrant
    *   MongoDB
    *   Neo4j (as seen in `agent/src/memory/store/neo4j_driver_adapter.go`)

---

### **3. Tool System & UTCP**

Lattice-Agent features a robust and extensible tool system built with the **Universal Tool Calling Protocol (UTCP)** in mind.

*   **`agent.Tool` Interface**: Developers can create custom tools by implementing a simple interface with two methods: `Spec()` and `Invoke()`.
*   **UTCP-Native**: As a steward of UTCP, the framework is designed for this protocol. UTCP is a standard that enables AI agents to discover and call tools regardless of the underlying transport, promoting interoperability.
*   **Model Agnostic**: The tool-calling mechanism works across different LLM provider APIs.

**Example Tool from `agent/README.md`:**
```go
// EchoTool repeats the provided input.
type EchoTool struct{}

func (e *EchoTool) Spec() agent.ToolSpec {
    return agent.ToolSpec{
        Name:        "echo",
        Description: "Echoes the provided text back to the caller.",
        InputSchema: map[string]any{ /* ... schema ... */ },
    }
}

func (e *EchoTool) Invoke(_ context.Context, req agent.ToolRequest) (agent.ToolResponse, error) {
    // ... implementation ...
}
```

---

### **4. LLM Integration**

The framework provides a unified `models.Agent` interface, allowing for pluggable LLM providers. This decouples the agent's logic from any specific model vendor.

*   **Supported Providers**: Adapters are available for Gemini, Anthropic, Ollama, and OpenAI.
*   **Bring Your Own**: The interface makes it straightforward to add support for custom or self-hosted models.

**Interface from `agent/src/models/interface.go`:**
```go
type Agent interface {
	Generate(context.Context, string) (any, error)
	GenerateWithFiles(context.Context, string, []File) (any, error)
}
```

---

### **5. Prerequisites & Configuration**

*   **Go Version**: 1.22+ (1.25 recommended).
*   **Database**: Optional, but PostgreSQL 15+ with the `pgvector` extension is recommended for persistent memory.
*   **Configuration**: Primarily through environment variables (e.g., `GOOGLE_API_KEY`, `DATABASE_URL`).

---

This summary covers the core specifications of the `go-agent` framework as detailed in the project files. It is a powerful tool for building sophisticated, production-grade AI systems in Go.# spec