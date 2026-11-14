# Assignment 2: Course Chat (LangGraph tools + Gradio)

The goal of this assignment is to design and implement an AI system with a conversational interface.

This notebook follows the structure of the project readme:

1. **Setup**
2. **Instructions prompt** (tone, restrictions and system-prompt protection)
3. **Service 1: API calls** (cat facts, dog facts, horoscope)
4. **Service 2: Semantic query** (Pitchfork reviews in ChromaDB)
5. **Service 3: Your choice** (not implemented in this version)
6. **User interface** (the LangGraph agent and the Gradio chat)
7. **Guardrails check**
8. **Decisions, limitations and possible improvements**

This implementation is based on LangGraph's tools. In the original project the model calls lived in `main.py`, the tools in the `tools_*.py` files, the prompt in `prompts.py` and the Gradio app in `app.py`. Here they are combined into one notebook so the whole system can be read and run top to bottom.

**Before running:** the ChromaDB and Postgres containers from the course (Docker) must be running, because Service 2 reads from them. The notebook still loads if they are not, and only the music tool reports an error.

## 1. Setup

**Decisions**
- Keys are read with `python-dotenv` from `.env` or `.secrets`. I try a few likely locations, so the notebook works from either `05_src` or a sub-folder.
- The original code imported a project logger (`utils.logger`). I use it when available and fall back to Python's `logging` otherwise.
- Environment variables are loaded **before** any client is created. In the original `app.py`, `load_dotenv` ran after the graph was built.

## 2. Instructions prompt

All restrictions and tone requirements live in one instructions prompt. It is kept in a function so it stays separate from the code that uses it.

**What it controls**
- **Cats and dogs:** the reply may not use the words *cat*, *dog* (or variations); *feline* and *canine* are used instead.
- **Music recommendations:** albums must come from the tool's database, include text based on the review, the artist, the release year and the score.
- **Taylor Swift:** she is never named ("she who shall not be named"), and only the Pitchfork score and release year are reported.
- **Horoscopes:** the reply starts by saying zodiac-based horoscopes cannot be given, attributes the reading to a fictional tradition, avoids the words "horoscope" and zodiac sign names, and ends with "Wink, wink."
- **Tone:** friendly, witty, Chicano-style Spanglish.
- **System prompt:** never revealed and never overridden. If asked, the reply is "No puedo decirte eso, carnal."

## 3. Service 1: API calls

There are a few API calls implemented throughout the course: cat facts, dog facts and a daily horoscope.

**Decisions**
- Each API call is wrapped as a LangChain **tool** with `@tool`. The docstring is what the model reads to decide when and how to call it, so the docstrings are written carefully (for example the horoscope tool lists the accepted signs and date formats).
- Each tool returns plain text. The instructions prompt then makes the model rewrite it in the chat's tone, so API output is not passed to the user verbatim.
- A request timeout and a simple error message were added, so a slow or broken API returns a readable message to the model instead of crashing the chat.

## 4. Service 2: Semantic query

This service is based on the Pitchfork exercise from class. A user question is matched **by meaning** against album reviews stored in ChromaDB, and structured data (score, artist, title) is added from the Postgres tables.

**How it works**
1. The query is embedded with `text-embedding-3-small` and ChromaDB returns the closest review passages (`recommend_albums`).
2. The review id is taken from each Chroma id (`<reviewid>_<chunk>`).
3. `additional_details` looks up title, artist and score in the `reviews` table.
4. Each result is returned as a `MusicReviewData` object, with a docstring that explains the score scale to the model (above 8.0 is a must-listen, above 6.5 is good).

**Decisions**
- The database lookup uses a **parameterised query**, not an f-string, so a review id can never change the SQL.
- Connecting to ChromaDB is wrapped so the notebook still loads if the container is down. The tool then returns a clear message.
- **Prerequisite:** the Docker versions of ChromaDB and Postgres must be running.

## 5. Service 3: Your choice

Not implemented in this version. The assignment allows function calling, web search or an MCP server connection for this service; it can be added as one more tool in the `tools` list in the next section.

## 6. User interface

### The agent (LangGraph)
The chat is a small LangGraph graph:
- `call_model` sends the instructions prompt plus the conversation to the model, with all tools bound. The model decides whether to answer or call a tool.
- `ToolNode` runs the requested tool and `tools_condition` is the standard stopping criterion. If the last message has tool calls the flow goes to the tools node; otherwise it ends.
- After a tool runs, the flow returns to `call_model`, so the model can turn the tool output into the final answer in the required tone.

### The chat window (Gradio)
- The conversational style (friendly, witty, Spanglish) comes from the instructions prompt.
- **Memory:** Gradio passes the full chat history on every turn. It is converted to LangChain messages, so the model remembers the entire conversation. The instructions are added fresh on every call and are never taken from the user, so a user cannot rewrite them.

## 7. Guardrails check

The guardrails are written into the instructions prompt (section 2). The cells below send test messages straight through `course_chat` so the behaviour can be checked and shown. Read each answer against the rule it tests.

## 8. Decisions, limitations and possible improvements

**Decisions**
- **LangGraph tools** keep each service in its own function and let the model choose which to call, with `tools_condition` as the standard stopping rule.
- **Prompt-based rules** keep the tone and restrictions in one editable place, separate from the code.
- **Structured outputs** (`MusicReviewData`) give the model clean fields (artist, review, score) so it can write a faithful recommendation.
- **Safer defaults than a first draft:** keys are loaded before use, SQL is parameterised, API calls have timeouts and error messages, and services that are down fail gracefully.

**Limitations and possible improvements**
- Service 3 is not implemented yet.
- The guardrails rely on the prompt alone. A code-level input filter (refuse restricted topics and prompt-extraction attempts before the model is called) and an output check would be harder to bypass.
- The full history is sent each turn. A summary or trimming strategy would handle very long conversations.
- Service 2 depends on the Docker ChromaDB and Postgres instances. A file-persisted ChromaDB with a CSV back end would be easier to share.
- API services (cat, dog, horoscope) are free third-party endpoints and can be slow or down.