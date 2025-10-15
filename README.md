# Football Transfer Market (Quarkus + Ollama)

> A simple football transfer simulation built with **Quarkus**, an **H2 in-memory database**, and **Ollama** for natural-language querying.

---

## 🏗️ Project Overview

This project exposes a small RESTful API that simulates a basic football transfer market.

You can:

- Create clubs with budgets (`/clubs`)
- Create players with prices and assign them to clubs (`/players`)
- Transfer players between clubs (`/transfers`)
- Ask natural-language questions to query the data (`/chat`), powered by an **Ollama LLM**

### ⚙️ How it works

1. **Entities and Persistence**

   - `Club` and `Player` are JPA entities managed by **Panache ORM**.
   - Clubs have a budget and own players.
   - The in-memory **H2** database stores all data during runtime.

2. **REST Endpoints**

   - Implemented using **JAX-RS** (`@Path`, `@GET`, `@POST`, etc.).
   - CRUD operations for clubs and players.
   - Transfers are handled transactionally via `TransferService`.

3. **Transfer Logic**

   - Buyer pays seller the player’s price.
   - Budgets are updated atomically within a single transaction.
   - Ownership (`player.club`) changes upon successful transfer.

4. **Chat + LLM Integration**
   - The `/chat` endpoint accepts a question.
   - The LLM (via **Ollama**) generates a SQL plan or direct answer.
   - The SQL is validated for safety, executed, and the results are fed back to the model.
   - The LLM verbalises the results into a natural language answer.

---

## 🧠 What I Learned (Quarkus)

1. **Panache and Simplicity**

   - Using `PanacheEntity` simplified CRUD operations like `listAll()` and `findByIdOptional()`.
   - It abstracts away repetitive JPA boilerplate and lets the focus stay on logic.

2. **`@Blocking` for JPA Calls**

   - JPA is blocking; marking resource classes with `@Blocking` ensures operations run on worker threads instead of the event loop.

3. **Transaction Management**

   - Applying `@Transactional` guarantees that multiple database updates (e.g., money transfer + ownership update) happen atomically.

4. **Controlling JSON Output**

   - Bidirectional relationships between `Club` and `Player` can cause infinite recursion.
   - Using `@JsonIgnore` on `Player.club` and exposing a computed `clubId` avoids this and keeps JSON responses clean.

5. **Centralised Error Handling**
   - The `GenericExceptionMapper` ensures consistent JSON error responses for `400`, `404`, and `500` errors without cluttering the resource classes.

---

## 🤖 What I Learned (Ollama Integration)

1. **Structured Prompts**

   - The **schema card** (`Prompts.schemaCard()`) forces the model to output only two valid JSON shapes:
     ```json
     {"type":"sql","sql":"..."}
     {"type":"final","answer":"..."}
     ```
   - Defining this contract made the model’s responses far more reliable.

2. **Two-Step LLM Flow**

   - Step 1: The model decides the plan (SQL or direct answer).
   - Step 2: If SQL is produced, it’s executed safely and results are verbalised by the model.
   - This chaining reduces prompt complexity and isolates errors clearly.

3. **SQL Safety**

   - `SqlRunner` validates that the SQL is **read-only** and adds a `LIMIT 200` automatically.
   - This prevents accidental data modification or large unbounded queries.

4. **Robust Response Parsing**

   - The `OllamaClient` extracts the model’s response safely and handles different response shapes returned by various Ollama versions.

5. **Configurable and Local**
   - The Ollama model name, base URL, and timeouts are configurable via `@ConfigProperty`.
   - This flexibility allows switching between local models or configurations without changing code.

---

## 🧩 What I Found Difficult

1. **Avoiding Infinite JSON Recursion**

   - The bidirectional `Club ↔ Player` mapping caused issues when serialising entities.
   - Using `@JsonIgnore` and computed IDs solved it cleanly.

2. **Transaction Boundaries**

   - Early tests showed inconsistent updates when not wrapped in a transaction.
   - Marking transfer operations with `@Transactional` ensured atomic commits.

3. **Safe LLM-Generated SQL**

   - Allowing a model to write SQL was risky.
   - I had to design guardrails:
     - Restrict SQL to `SELECT` only.
     - Limit results to 200 rows.
     - Define schema explicitly in the prompt.

4. **Handling Ollama Responses**

   - Ollama’s response format changed slightly across versions.
   - Adding defensive parsing avoided null pointers and made error messages clearer.

5. **H2 and Column Metadata**

   - Native queries sometimes returned results without column names.
   - I worked around this by inferring column widths when metadata wasn’t available.

6. **Lazy Loading and JSON**
   - Returning entities with `FetchType.LAZY` can cause exceptions after transactions.
   - Ignoring lazy fields in JSON output prevented this problem.

---

## 🧩 Key Classes and Their Roles

| Class                      | Purpose                                                            |
| -------------------------- | ------------------------------------------------------------------ |
| **ClubResource**           | REST endpoints for clubs (create, list, get).                      |
| **PlayerResource**         | REST endpoints for players (create, list, get).                    |
| **TransferResource**       | Orchestrates player transfers using `TransferService`.             |
| **TransferService**        | Core business logic for transferring players and budgets.          |
| **ChatResource**           | LLM orchestration: schema prompt → SQL plan → execute → verbalise. |
| **OllamaClient**           | Handles HTTP communication with the Ollama API.                    |
| **SqlRunner**              | Runs native queries safely with read-only validation.              |
| **Prompts**                | Defines schema and verbalisation instructions for the LLM.         |
| **GenericExceptionMapper** | Converts exceptions into clean JSON error responses.               |

---

## 🧪 Example cURL Commands

```bash
# Create a club
curl -s -X POST http://localhost:8080/clubs \
  -H 'Content-Type: application/json' \
  -d '{"name":"Barcelona","budget":200000000.00}'

# Create a player (owned by club id 1)
curl -s -X POST http://localhost:8080/players \
  -H 'Content-Type: application/json' \
  -d '{"name":"Pedri","price":75000000.00,"clubId":1}'

# Transfer the player to buyer club id 2
curl -s -X POST http://localhost:8080/transfers \
  -H 'Content-Type: application/json' \
  -d '{"buyerClubId":2,"playerId":1}'

# Ask a natural language question via Ollama
curl -s -X POST http://localhost:8080/chat \
  -H 'Content-Type: application/json' \
  -d '{"question":"list the players in barcelona"}'
```
