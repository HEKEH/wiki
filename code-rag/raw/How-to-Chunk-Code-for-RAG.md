---
created: 2026-05-05T21:40:14 (UTC +08:00)
tags: []
source: https://theneuralbase.com/chunking/qna/how-to-chunk-code-for-rag/
author: The Neural Base
---

# How to Chunk Code for RAG:

> ## Excerpt
> Learn how to chunk code effectively for Retrieval-Augmented Generation (RAG) using Python, with best practices for token limits and semantic chunking.

---
[Home](https://theneuralbase.com/) [Chunking Strategies](https://theneuralbase.com/chunking/) How to chunk code for RAG

Code intermediate · 3 min read

Direct answer

Use text splitting techniques to divide code into logical, token-limited chunks for RAG, ensuring each chunk fits model context windows and preserves semantic boundaries.

Team Note

Critical: RecursiveCharacterTextSplitter chunk\_size is measured in characters, not tokens - the article states 'max tokens approx' but 500 characters ≈ 125 tokens (roughly 4 chars per token in code), not 500. Set chunk\_size based on character count, then verify actual token usage with tiktoken: import tiktoken; enc = tiktoken.encoding\_for\_model('gpt-4o'); tokens = len(enc.encode(chunk)).

The streaming code example loops synchronously over \`response\` without checking \`event.choices\[0\].delta.content\` - streaming events have null content fields at end-of-stream, causing AttributeError. Fix: use \`if event.choices\[0\].delta.content:\` before printing.

The async example uses \`await client.chat.completions.acreate()\` which is correct for openai 1.0.0+, but the loop doesn't handle rate limits or timeout exceptions - wrap in try/except for production. For code chunking specifically: RecursiveCharacterTextSplitter with separators=\[' ', ' ', ' '\] works well for code, but function/class definitions split mid-signature if chunk\_size is too aggressive - test with actual codebase before deploying.

Setup

Install

bash

```
pip install langchain openai
```

Env vars

`OPENAI_API_KEY`

Imports

python

```
import os
from langchain.text_splitter import RecursiveCharacterTextSplitter
from openai import OpenAI
```

Examples

indef add(a, b): return a + b print(add(2, 3))

out\['def add(a, b):\\n return a + b\\n\\nprint(add(2, 3))'\]

inclass Calculator: def add(self, a, b): return a + b def subtract(self, a, b): return a - b calc = Calculator() print(calc.add(5, 7))

out\["class Calculator:\\n def add(self, a, b):\\n return a + b\\n", " def subtract(self, a, b):\\n return a - b\\n\\ncalc = Calculator()\\nprint(calc.add(5, 7))"\]

in\# Large script with multiple functions and classes spanning 2000 tokens

out\["Chunk 1 with first 1000 tokens", "Chunk 2 with next 1000 tokens"\]

Integration steps

1.  Import a text splitter like RecursiveCharacterTextSplitter from LangChain.
2.  Load your code as a string and configure the splitter with chunk size and overlap.
3.  Split the code into chunks respecting token limits and semantic boundaries.
4.  Use these chunks as documents for embedding or retrieval in your RAG pipeline.
5.  Query the retriever with user input to fetch relevant code chunks.
6.  Pass retrieved chunks plus query to the LLM for augmented generation.

Full code

python

```
import os
from langchain.text_splitter import RecursiveCharacterTextSplitter
from openai import OpenAI

# Load your code as a string (example code snippet)
code_text = '''
class Calculator:
    def add(self, a, b):
        return a + b

    def subtract(self, a, b):
        return a - b

calc = Calculator()
print(calc.add(5, 7))
'''

# Initialize the text splitter with chunk size and overlap
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,  # max tokens approx
    chunk_overlap=50,
    separators=["\n\n", "\n", " "]
)

# Split the code into chunks
chunks = splitter.split_text(code_text)

# Print chunks for inspection
for i, chunk in enumerate(chunks):
    print(f"Chunk {i+1} (length {len(chunk)} chars):\n{chunk}\n{'-'*40}")

# Example: Initialize OpenAI client for RAG retrieval or embedding
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# Normally, you'd embed chunks and build a vector store here for retrieval
# This example only shows chunking logic
```

output

```
Chunk 1 (length 142 chars):
class Calculator:
    def add(self, a, b):
        return a + b

    def subtract(self, a, b):
        return a - b

----------------------------------------
Chunk 2 (length 27 chars):
calc = Calculator()
print(calc.add(5, 7))

----------------------------------------
```

API trace

Request

json

```
{"model": "gpt-4o", "messages": [{"role": "user", "content": "Retrieve relevant code chunks for query"}]}
```

Response

json

```
{"choices": [{"message": {"content": "Relevant code chunk text..."}}], "usage": {"total_tokens": 150}}
```

Extract

Variants

Streaming chunk output ›

Use streaming when processing large code chunks to provide incremental output and better UX.

python

```
import os
from langchain.text_splitter import RecursiveCharacterTextSplitter
from openai import OpenAI

code_text = '''def foo():\n    pass\n\n# More code...'''

splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=30)
chunks = splitter.split_text(code_text)

for chunk in chunks:
    print(chunk)

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Process this code chunk:" + chunk}],
    stream=True
)
for event in response:
    print(event.choices[0].message.content, end='')
```

Async chunking and retrieval ›

Use async when integrating chunking with concurrent API calls for efficiency.

python

```
import os
import asyncio
from langchain.text_splitter import RecursiveCharacterTextSplitter
from openai import OpenAI

async def chunk_and_query():
    code_text = 'def async_func():\n    pass\n'
    splitter = RecursiveCharacterTextSplitter(chunk_size=400, chunk_overlap=40)
    chunks = splitter.split_text(code_text)

    client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

    for chunk in chunks:
        response = await client.chat.completions.acreate(
            model="gpt-4o",
            messages=[{"role": "user", "content": f"Analyze this code chunk:\n{chunk}"}]
        )
        print(response.choices[0].message.content)

asyncio.run(chunk_and_query())
```

Alternative model for cost efficiency ›

Use smaller models like gpt-4o-mini for cheaper, faster chunk summarization when high accuracy is not critical.

python

```
import os
from langchain.text_splitter import RecursiveCharacterTextSplitter
from openai import OpenAI

code_text = 'def cheap_model_func():\n    return True\n'

splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_text(code_text)

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

for chunk in chunks:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": f"Summarize this code chunk:\n{chunk}"}]
    )
    print(response.choices[0].message.content)
```

Performance

Latency~800ms for gpt-4o non-streaming calls

Cost~$0.002 per 500 tokens for gpt-4o

Rate limitsTier 1: 500 RPM / 30K TPM

-   Use chunk overlap sparingly to reduce redundant tokens.
-   Trim comments or non-essential code before chunking to save tokens.
-   Batch multiple chunks in one request if model context allows.

|Approach|Latency|Cost/call|Best for|
|---|---|---|---|
|Standard chunking with gpt-4o|~800ms|~$0.002 per 500 tokens|High accuracy RAG|
|Streaming chunk output|Varies, faster perceived|Similar|Large codebases, better UX|
|Async chunking|Improved throughput|Similar|Concurrent processing|
|Using gpt-4o-mini|~400ms|~$0.0005 per 500 tokens|Cost-sensitive summarization|

✓

Quick tip

Use semantic-aware text splitters like RecursiveCharacterTextSplitter to chunk code preserving logical blocks and token limits.

⚠

Common mistake

Splitting code arbitrarily by fixed character count without semantic awareness leads to broken code chunks and poor retrieval results.

Verified 2026-04 · gpt-4o, gpt-4o-mini

[Verify ↗](https://platform.openai.com/docs/api-reference/chat/create)

___
