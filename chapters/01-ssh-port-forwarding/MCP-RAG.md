# Technical Documentation: MCP & Graph RAG Ecosystem

## 1. Architectural Overview: The MCP & Graph RAG
The system transition focuses on an agentic ecosystem where the **Model Context Protocol (MCP)** acts as the primary interface between the LLM and specialized infrastructure tools.

*   **Hybrid RAG Strategy:** 
    *   **Chroma DB:** A semantic vector database used for vectorized tokens (embeddings).
    *   **Neo4j:** A graph database used to map structured relationships, environment variables, and job dependencies.
*   **The Ingestion Pipeline:** Semantic transformers ingest code snippets and documentation into an embedding space organized into "communities of interest" using **K-means clustering**.
*   **The Gateway:** A Docker-based gateway (running on port `1888`) handles REST endpoints, enabling the LLM to execute reasoning steps by querying the integrated RAG and Graph datasets.

## 2. Semantic Encoding & Documentation Standards (RST)
The project utilizes **reStructuredText (RST)** with custom directives to guide LLM reasoning directly from the documentation.

*   **Context Discriminators:** Specific tags that help the LLM differentiate between similar concepts or resources.
*   **Directive Types:**
    *   **Intent Designators:** Explicitly define the purpose of a code block.
    *   **Compliance Designators:** Highlight mandatory security or regulatory rules.
    *   **Anti-Pattern Designators:** Identify "what not to do," preventing the LLM from suggesting fragile or procedural ("iffy-diffy") solutions.
*   **Logic Shift:** Moves development from procedural (if-then) logic toward rules-based guidance that the LLM consumes for automated code generation.

## 3. Operational Parity & Gap Analysis
 Parity between the **COTS (Commodity Off-The-Shelf)** local environment and **AWS Production** is maintained through automated diagnostics.

*   **Drift Detection:** Daily automated health checks identify discrepancies between local Docker containers and AWS instances.
*   **Automated Remediation:** When "gaps" are found, an agent generates a **System Design Document (SDD)**. Tools like the **Kira CLI** are then used to push remediation requirements to the target platform.
*   **Platform Awareness:** Logic is handled via case statements or factory patterns to account for platform-specific variables (Docker vs. AWS).

## 4. Infrastructure, Security & Connectivity
The workflow has moved away from public-facing "Development Tunnels" to a hardened **SSH-based architecture**.

*   **SSH Tunneling:** Remote service ports (e.g., port `1888` for MCP Gateway, port `8080` for Chroma DB) are forwarded to `localhost` on the developer's GFE.
*   **Passwordless Authentication:** Access is managed via RSA keys and the `.ssh/config` file.
*   **MDA (Multi-Domain Authentication):** CLI tools utilize browser-based MDA to link sessions to corporate identities, replacing static IAM keys.

## 5. Multi-User Workflow & Steering
*   **Real-Time Steering:** The ability to provide guidance to the LLM during reasoning (e.g., "Use the COTS SDD") to correct logic in real-time.
*   **Multi-Tenancy:** Each developer or feature set operates within a "tenant" collection, allowing multiple users to engage with the same RAG.
*   **Git Strategy:** Features are pushed to a **Community Server** (Development) and promoted to the **Licensed Server** (Production/Gold Copy) via Configuration Manager (CM) approved Merge Requests.

---

## Further Reading & Research Topics

### Semantic Documentation & Theory
*   **Advanced RST Directives:** Extending reStructuredText for metadata tagging and semantic ingestion.
*   **Context Discrimination:** Study of discriminative vs. generative modeling for intent recognition.
*   **Anti-Pattern Theory:** Research on failure modes in agentic workflows when attempting declarative logic.

### Data & Retrieval Science
*   **Graph RAG Algorithms:** Investigating the convolution of Neo4j and Chroma DB for higher retrieval accuracy.
*   **Clustering in High-Dimensional Spaces:** Study of k-means and "minimal surface" algorithms for vector databases.
*   **Linear Algebra for Embeddings:** Concepts of "span" and "orthonormal sets" in semantic vector spaces.

### Infrastructure & Orchestration
*   **Model Context Protocol (MCP) Spec:** Detailed study of the MCP standard for JSON-based tool definitions.
*   **Docker Volume Mapping:** Best practices for persistent RAG storage in containerized environments.
*   **Platform Parity Logic:** implementing factory patterns for multi-cloud (AWS/On-prem) parity.

### LLM Reasoning
*   **Chain of Thought (CoT) Engineering:** Techniques for improving reasoning steps in language models.
*   **Semantic Transformers:** Internal mechanisms for vectorized token generation from raw documentation.
*   **PCA in Semantics:** Using Principal Component Analysis to optimize context window efficiency.

