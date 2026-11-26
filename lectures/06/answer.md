# Lecture 06 answers

## Exercises

<!-- Paste the original [tool call] lines. Leave unfinished steps blank. -->

- Tool name in STEP 1 — glossary_search_glossary
- STEP 2 [tool call] — glossary_search_glossary({'query': 'homework'})
- STEP 3 [tool call] — fetch_fetch({'url': 'https://modelcontextprotocol.io/'})
- STEP 4 [tool call] — aws_aws___search_documentation({'search_phrase': 'Amazon S3 versioning overview', 'topics': ['general']})
- Observations and questions —
  - With all three servers configured, `--list-tools` showed 7 tools: `glossary_search_glossary`, `fetch_fetch`, and five `aws_aws___…` tools (`search_documentation`, `read_documentation`, `list_regions`, `get_regional_availability`, …). The server-name prefix keeps tools with the same name apart.
  - Adding a tool only took a config change. `glossary` and `fetch` both start as processes on my machine (`command`). `fetch` is downloaded by `uvx` at a pinned version. `aws` runs on AWS's servers, and the agent connects to it only by `url`, so my tool inputs are sent to AWS.
  - The `fetch_fetch` description says "Although originally you did not have internet access … this tool now grants you internet access." The model reads this text exactly as the server's author wrote it, so a third-party server can steer the model through its tool descriptions. This is a reason to check who provides a server before adding it.
  - In STEP 3, the fetch server first checked `robots.txt`, followed a 307 redirect to `/docs/…/intro.md`, and fetched the page. The model call that came next failed once with "The model is busy (503)". That was a temporary error from the model provider, not from the tool, and the same command worked when I ran it again.
  - All three questions used exactly 2 model responses (about 5,500–6,300 input tokens). In the first response the model chose the tool and its arguments, and in the second it answered from the tool's result. The model picked the right server each time from the tool descriptions alone. For STEP 4 it used `search_documentation` and did not need `read_documentation`, which matches what the AWS tool description tells it to do.
  - In STEP 2 the model searched the shorter query `'homework'` instead of the whole question, and it still found the "3 homework levels" entry.

## Homework

<!-- Complete one of the Basic, Implementation, or Advanced levels. -->

- Level attempted (Basic / Implementation / Advanced) — Implementation
- What you did —
  - Ported `make_quiz`, which I wrote in Lecture 03 in my Japanese-repo fork, into `app/tools.py` with an English docstring and messages. I also ported its 4 tests and registered it in `app/agent.py`.
  - Exposed it on the glossary MCP server with `server.tool()(tools.make_quiz)` in `app/mcp_server.py`. `--list-tools` showed `glossary_make_quiz`.
  - For a lecture with no glossary terms, it returns the message "There are no glossary terms for Lecture N. Lectures with quizzes: 1, 2, 3, 4, 6".
  - Asked "Make a quiz for Lecture 99." twice: once with that message sent by `return`, and once with the same message sent by `raise ValueError(...)`. Afterwards I restored `return`, and `uv run pytest -q` passed (25 tests).
- Results —

  | | `return` | `raise` |
  |---|---|---|
  | Trace | `[tool return] glossary_make_quiz -> There are no glossary terms for Lecture 99. Lectures with quizzes: 1, 2, 3, 4, 6` | `[tool error]  glossary_make_quiz -> Error executing tool make_quiz` |
  | Model's answer | "There is no Lecture 99 … The available lectures with quizzes and glossary terms in this course are Lectures 1, 2, 3, 4, and 6. Would you like a review quiz for one of those lectures instead?" | "There is no Lecture 99 available in the course glossary … Please choose an existing lecture number for your quiz!" |
  | Usage | input 5,461 / output 87 tokens, 2 model responses | input 5,444 / output 52 tokens, 2 model responses |

- Observations and questions —
  - With `raise`, my message did not reach the model. The MCP server logged the full `ValueError` in my terminal but sent the model only "Error executing tool make_quiz". The model could guess that Lecture 99 doesn't exist, because that was in my question. It could not name the lectures that do exist, so it could only say "choose an existing lecture number".
  - With `return`, the model got the list of available lectures and suggested a quiz for one of them, so the answer was more useful.
  - In this run Gemini did not retry after the error (2 model responses in both cases). In a test with a fake model that always repeats the same call, the `raise` version retried, then stopped with "exceeded max retries count of 1". So `raise` can also cost extra model calls or end the run.
  - Conclusion: use `return` with a clear message for expected problems the model can act on, such as a bad argument. Keep `raise` for real failures, such as a missing file. Question: when a tool raises an error, should the MCP server send the error message to the model, or is hiding it safer because messages could leak internal details?
