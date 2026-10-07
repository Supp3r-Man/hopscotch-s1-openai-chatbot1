# Hopscotch Support Chatbot - Session 1: Raw OpenAI SDK

A return-policy customer support chatbot for **Hopscotch** (premium kids' fashion, Mumbai),
built the "before LangChain" way: the plain `openai` SDK plus a little glue code.

## Series

| Session | Repo | Topics |
|---|---|---|
| **1 - you are here** | [hopscotch-s1-openai-chatbot](https://github.com/niti007/hopscotch-s1-openai-chatbot) | Raw OpenAI SDK, message list as memory, glue code |
| 2 | [hopscotch-s2-langchain-chatbot](https://github.com/niti007/hopscotch-s2-langchain-chatbot) | LangChain models, messages, prompt templates, chains, chat history with `session_id` |
| 3 | [hopscotch-s3-lcel-chatbot](https://github.com/niti007/hopscotch-s3-lcel-chatbot) | Runnables, LCEL, `RunnableParallel`, `RunnableBranch`, streaming |

## What this session teaches

- An LLM is **stateless**: it remembers nothing between calls.
- "Memory" is just a **Python list of `{"role", "content"}` dicts** that we resend in full every turn.
- A **system prompt** (role + policy document + rules) steers behaviour; here it is built with an f-string.
- The terminal `while True: input()` chat loop maps onto Streamlit, which re-runs the whole script on every message (`st.session_state` keeps the list alive).
- Safe escalation: Section 4 of the policy (child safety, allergies, orders over INR 5,000, unclear facts) must go to a human, and the prompt tells the model so.

## Architecture

```
User
  |  types a message
  v
Streamlit (app.py re-runs top to bottom)
  |  append {"role": "user", ...}
  v
messages list  (st.session_state.messages, system prompt first)
  |  whole list sent every turn
  v
openai SDK  client.chat.completions.create(...)
  |  base_url = https://openrouter.ai/api/v1
  v
OpenRouter
  |  routes to
  v
openai/gpt-4o-mini  -> reply -> appended as {"role": "assistant", ...}
```

## Setup

Python 3.10+ required.

```bash
uv venv --python 3.11 && source .venv/bin/activate && uv pip install -r requirements.txt
```

Plain pip alternative:

```bash
python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt
```

Then add your key (free signup, get one at https://openrouter.ai/keys):

```bash
cp .env.example .env
# edit .env and paste your OPENROUTER_API_KEY
```

## Run

```bash
streamlit run app.py
```

## Things to try in class

1. `I bought a dress 10 days ago, never worn, tags on - can I return it?` (standard return, should be approved)
2. `The sole of my son's sneakers peeled off after 3 weeks of school` (defect: should ask for a photo)
3. `A button came off and my toddler nearly put it in his mouth` (child safety: human agent)
4. `My order was ₹7,200, I want a refund` (over INR 5,000: human agent)
5. Ask a follow-up like "what about the shoes?" and open the sidebar expander: watch the list grow and see the whole thing resent.
6. Click **Clear chat** and repeat the follow-up: the bot has forgotten everything. That is the memory list being reset.
7. Edit the rules in `SYSTEM_PROMPT` (e.g. remove "answer only from the policy") and see what changes.

## Limitations -> why Session 2

- **Tied to one SDK and wire format.** `client.chat.completions.create(...)` and `response.choices[0].message.content` are OpenAI-specific.
- **Model string is hardcoded** (`"openai/gpt-4o-mini"`).
- **Messages are raw dicts.** No types, no validation; a typo in `"role"` fails at runtime.
- **The prompt is an f-string** glued together with the policy text: no reusable template, no input variables.
- **History is one list per browser tab.** There is no session abstraction (no `session_id`, no storage, no way to resume).
- **Switching providers means a rewrite** of the client, the message format and the response parsing. For example, going to Claude's direct SDK:

```python
# Today (OpenAI SDK)
client = OpenAI(api_key=key, base_url="https://openrouter.ai/api/v1")
messages = [{"role": "system", "content": SYSTEM_PROMPT}, {"role": "user", "content": text}]
resp = client.chat.completions.create(model="openai/gpt-4o-mini", messages=messages)
reply = resp.choices[0].message.content

# Anthropic SDK: different client, system prompt is a separate argument,
# max_tokens is required, and the reply lives somewhere else
client = anthropic.Anthropic(api_key=key)
messages = [{"role": "user", "content": text}]            # no "system" role in the list
resp = client.messages.create(model="claude-...", system=SYSTEM_PROMPT,
                              messages=messages, max_tokens=1024)
reply = resp.content[0].text
```

Session 2 fixes these with LangChain's model abstraction, message classes, prompt templates, chains and session-based chat history:
[hopscotch-s2-langchain-chatbot](https://github.com/niti007/hopscotch-s2-langchain-chatbot).