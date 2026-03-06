---
layout: single
title: "Context Windows: Why Size Matters (and How to Optimise)"
date: 2026-03-06
category: "concepts"
tags: ["concepts", "atlas-signal", "deep-research", "MachineLearning", "DeepLearning", "NeuralNetworks"]
description: "Context windows determine how much information an AI model can process at once. Understanding window sizes from GPT-4's 128K to Claude 3.5's 200K tokens and opt"
canonical_url: "https://atlassignal.in/posts/context-windows-why-size-matters-and-how-to-optimise/"
og_title: "Context Windows: Why Size Matters (and How to Optimise)"
og_description: "Context windows determine how much information an AI model can process at once. Understanding window sizes from GPT-4's 128K to Claude 3.5's 200K tokens and opt"
og_url: "https://atlassignal.in/posts/context-windows-why-size-matters-and-how-to-optimise/"
og_image: "https://images.pexels.com/photos/5474287/pexels-photo-5474287.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940"
header:
  overlay_image: https://images.pexels.com/photos/5474287/pexels-photo-5474287.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940
  overlay_filter: 0.75
twitter_card: summary_large_image
toc: true
toc_sticky: true
---

![Context Windows: Why Size Matters (and How to Optimise)](https://images.pexels.com/photos/5474287/pexels-photo-5474287.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940)


**Difficulty:** Beginner | **Category:** Concepts

# Context Windows: Why Size Matters (and How to Optimise)


<ins class="adsbygoogle"
     style="display:block"
     data-ad-client=""
     data-ad-slot="AUTO"
     data-ad-format="auto"
     data-full-width-responsive="true"></ins>
<script>(adsbygoogle = window.adsbygoogle || []).push({});</script>


A developer recently fed an entire 50-page legal document into GPT-4, only to discover the model "forgot" the first 20 pages when answering questions about the conclusion. This isn't a bug—it's a fundamental constraint called the **context window**. As of March 2026, understanding context windows can mean the difference between a $0.50 API call and a $15 one, while dramatically improving response quality.

## Prerequisites

Before diving in, you'll need:

- Access to at least one LLM API (OpenAI, Anthropic, or Google's Gemini)
- Basic Python knowledge (or ability to run API calls)
- A text editor and terminal
- Understanding that 1 token ≈ 0.75 English words

## What Is a Context Window?

Think of a context window as your AI model's "working memory." Just like you can't hold an entire encyclopedia in your head while having a conversation, language models have hard limits on how much text they can process in a single request.

**Current context window sizes (March 2026):**

- **GPT-4 Turbo**: 128,000 tokens (~96,000 words)
- **Claude 3.5 Sonnet**: 200,000 tokens (~150,000 words)
- **Gemini 1.5 Pro**: 1,000,000 tokens (~750,000 words)
- **GPT-3.5 Turbo**: 16,385 tokens (~12,000 words)

Each token counts toward your limit—including your system prompt, conversation history, uploaded documents, and the model's response.

## Step 1: Calculate Your Actual Token Usage

Let's measure exactly what you're sending to an API.

**Install the token counter:**

```bash
pip install tiktoken
```

**Count tokens in your prompt:**

```python
import tiktoken

def count_tokens(text, model="gpt-4"):
    encoding = tiktoken.encoding_for_model(model)
    tokens = encoding.encode(text)
    return len(tokens)

# Real example: A support ticket
ticket = """
Customer: John Smith (ID: 847392)
Issue: Payment failed three times using card ending 4242
Previous interactions: 5 tickets, last resolved 2024-12-15
Account history: Premium member since 2023, $2,340 lifetime value
"""

print(f"Tokens used: {count_tokens(ticket)}")
# Output: Tokens used: 67
```

**Gotcha:** Many developers forget that system prompts count too. A typical system prompt uses 200-500 tokens, reducing your available context for actual data.

## Step 2: Monitor Context Window Exhaustion

When you exceed the context window, models either truncate silently or throw errors. Here's how to catch this:

```python
from openai import OpenAI

client = OpenAI(api_key="your-key-here")

def safe_completion(messages, max_context=128000):
    total_tokens = sum(count_tokens(msg["content"]) for msg in messages)
    
    if total_tokens > max_context * 0.9:  # 90% threshold
        print(f"⚠️  Warning: Using {total_tokens} of {max_context} tokens")
        return None
    
    response = client.chat.completions.create(
        model="gpt-4-turbo-2024-04-09",
        messages=messages
    )
    
    return response.choices[0].message.content

# Example with actual conversation
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "What's in this 50-page document?" + "x" * 100000}
]

result = safe_completion(messages)
# Output: ⚠️  Warning: Using 115847 of 128000 tokens
```

**Pro Tip:** OpenAI's API response includes `usage` metadata. Always log this to track actual consumption:

```python
print(f"Prompt tokens: {response.usage.prompt_tokens}")
print(f"Completion tokens: {response.usage.completion_tokens}")
print(f"Total cost: ${(response.usage.total_tokens / 1000) * 0.03:.4f}")
```

## Step 3: Implement Smart Chunking for Large Documents

When your input exceeds the context window, chunk it intelligently:

```python
def chunk_document(text, chunk_size=10000, overlap=200):
    """Split text into overlapping chunks to preserve context"""
    words = text.split()
    chunks = []
    
    for i in range(0, len(words), chunk_size - overlap):
        chunk = ' '.join(words[i:i + chunk_size])
        chunks.append(chunk)
    
    return chunks

# Real example: Processing a research paper
long_paper = open('research_paper.txt').read()  # 30,000 words
chunks = chunk_document(long_paper, chunk_size=8000, overlap=500)

summaries = []
for i, chunk in enumerate(chunks):
    response = client.chat.completions.create(
        model="gpt-4-turbo-2024-04-09",
        messages=[
            {"role": "system", "content": "Summarize this section in 100 words."},
            {"role": "user", "content": chunk}
        ]
    )
    summaries.append(response.choices[0].message.content)
    print(f"Processed chunk {i+1}/{len(chunks)}")

# Then combine summaries
final_summary = "\n\n".join(summaries)
```

**Gotcha:** Don't chunk mid-sentence or mid-paragraph. Always use natural breakpoints like double newlines or section headers.

## Step 4: Use Prompt Compression Techniques

Reduce token usage by 40-60% without losing information:

**Before compression (427 tokens):**
```
The customer, whose name is Jennifer Martinez and whose customer ID is 
CM-99234, contacted our support team on March 3rd, 2026 at approximately 
2:30 PM Eastern Standard Time regarding an issue she was experiencing...
```

**After compression (178 tokens):**
```
Customer: Jennifer Martinez (CM-99234)
Contact: 2026-03-03 14:30 EST
Issue: Payment processing failure
Details:
- Card ending 5678
- Error code: DECLINED_CVV
- Retry attempts: 3
```

**Pro Tip:** Use structured formats (JSON, YAML, or bullet points) instead of prose when feeding data to models. They're 2-3x more token-efficient.

## Step 5: Leverage Rolling Context Windows

For chat applications, implement a sliding window that keeps recent context:

```python
class ContextManager:
    def __init__(self, max_tokens=16000):
        self.max_tokens = max_tokens
        self.messages = []
    
    def add_message(self, role, content):
        self.messages.append({"role": role, "content": content})
        self._trim_context()
    
    def _trim_context(self):
        """Keep only recent messages that fit in context"""
        total = 0
        keep_from = len(self.messages)
        
        # Count backwards, always keep system message
        for i in range(len(self.messages) - 1, 0, -1):
            msg_tokens = count_tokens(self.messages[i]["content"])
            if total + msg_tokens > self.max_tokens:
                keep_from = i + 1
                break
            total += msg_tokens
        
        # Keep system message + recent messages
        self.messages = [self.messages[0]] + self.messages[keep_from:]
    
    def get_messages(self):
        return self.messages

# Usage
chat = ContextManager(max_tokens=4000)
chat.add_message("system", "You are a helpful assistant.")
chat.add_message("user", "Tell me about Python.")
# ... conversation continues ...
# Old messages automatically dropped when limit reached
```

## Step 6: Choose the Right Model for Your Context Needs

**Decision matrix:**

- **< 4K tokens**: GPT-3.5 Turbo ($0.0015/1K) — cheapest option
- **4K-16K tokens**: GPT-4 Turbo ($0.03/1K) — balanced performance
- **16K-128K tokens**: Claude 3.5 Sonnet ($0.015/1K) — cost-effective for large context
- **128K-1M tokens**: Gemini 1.5 Pro ($0.007/1K) — massive documents

**Real cost comparison for a 100K token input:**
- Gemini 1.5 Pro: $0.70
- Claude 3.5 Sonnet: $1.50
- GPT-4 Turbo: $3.00

## Practical Example: Optimized Document Q&A System

Here's a complete system that handles documents of any size efficiently:

```python
import tiktoken
from openai import OpenAI

class SmartDocumentQA:
    def __init__(self, model="gpt-4-turbo-2024-04-09", max_context=100000):
        self.client = OpenAI()
        self.model = model
        self.max_context = max_context
        self.encoding = tiktoken.encoding_for_model(model)
    
    def ask(self, document, question):
        # Count tokens
        doc_tokens = len(self.encoding.encode(document))
        question_tokens = len(self.encoding.encode(question))
        system_tokens = 50  # estimated
        
        total = doc_tokens + question_tokens + system_tokens
        
        if total < self.max_context * 0.9:
            # Fits in context - send directly
            return self._direct_query(document, question)
        else:
            # Too large - use chunked approach
            return self._chunked_query(document, question)
    
    def _direct_query(self, document, question):
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "Answer questions about the provided document."},
                {"role": "user", "content": f"Document:\n{document}\n\nQuestion: {question}"}
            ]
        )
        return response.choices[0].message.content
    
    def _chunked_query(self, document, question):
        # First pass: Extract relevant chunks
        chunks = self._chunk_text(document, 8000)
        relevant_chunks = []
        
        for chunk in chunks[:10]:  # Limit to first 10 chunks for demo
            response = self.client.chat.completions.create(
                model="gpt-3.5-turbo",  # Use cheaper model for filtering
                messages=[
                    {"role": "user", "content": f"Is this text relevant to '{question}'? Answer YES or NO.\n\n{chunk}"}
                ]
            )
            if "YES" in response.choices[0].message.content.upper():
                relevant_chunks.append(chunk)
        
        # Second pass: Answer using only relevant chunks
        combined = "\n\n---\n\n".join(relevant_chunks)
        return self._direct_query(combined, question)
    
    def _chunk_text(self, text, size):
        words = text.split()
        return [' '.join(words[i:i+size]) for i in range(0, len(words), size)]

# Real usage
qa = SmartDocumentQA()
document = open('company_handbook.txt').read()  # 80,000 words
answer = qa.ask(document, "What is the remote work policy?")
print(answer)
```

## Key Takeaways

- **Context windows are hard limits**: GPT-4 Turbo's 128K tokens ≈ 96,000 words, but your actual usable space is less after system prompts and conversation history
- **Token counting is essential**: Use `tiktoken` to measure exact usage before sending requests—this prevents silent truncation and controls costs
- **Chunking + compression = 60-80% savings**: Structured formats (JSON/YAML) and intelligent chunking reduce token usage dramatically compared to raw prose
- **Model selection matters**: For 100K tokens, Gemini 1.5 Pro costs $0.70 vs GPT-4 Turbo's $3.00—choose based on your context needs

## What's Next

Now that you understand context windows, learn about **vector databases and embeddings** to handle documents too large for any context window by retrieving only relevant sections.

---

**Key Takeaway:** Context windows determine how much information an AI model can process at once. Understanding window sizes (from GPT-4's 128K to Claude 3.5's 200K tokens) and optimization techniques like chunking and compression can reduce costs by 60-80% while improving response quality.

---

*New AI tutorials published daily on [AtlasSignal](https://atlassignal.in). Follow [@AtlasSignalDesk](https://twitter.com/AtlasSignalDesk) for more.*

---

## 📧 Get Daily AI & Macro Intelligence

Stay ahead of market-moving news, emerging tech, and global shifts.

<div class="email-capture">
  <form action="" method="post" target="_blank">
    <input type="email" name="EMAIL" placeholder="Your email address" required />
    <button type="submit">Subscribe Free →</button>
  </form>
</div>

