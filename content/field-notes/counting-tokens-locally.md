---
title: "Counting Tokens Without Spending Any"
date: 2026-06-08
tags: ["ollama", "llm", "python", "agents"]
draft: false
---

I kept guessing at how much context my agent prompts were eating. Not a great way to run things.

The obvious fix is a token counter. The less obvious part is that you don't need an API to build one. If you're already running models locally, the tokenizer is right there.

So I wrote a tiny CLI. Point it at a file, name a model, get a count. That's the whole thing.

```bash
python token_counter.py agent.py --model gemma3:4b
```

It leans on Ollama, which means zero cost and zero keys. The counts come straight from the model's own tokenizer, so they're exact, not an estimate. Run it against two models at once and you can watch the same prompt land differently depending on who's counting.

```bash
python token_counter.py agent.py --model gemma3:4b qwen3.5:latest
```

![Same agent.py counted across two models](/agent-comparison.png)

Eighty-two tokens on Gemma, seventy-four on Qwen. Same file. That gap is the whole point.

It also takes piped input if you just want to check a quick prompt:

```bash
echo "your prompt here" | python token_counter.py --model gemma3:4b
```

![Counting a piped prompt](/stdin-example.png)

A small touch I'm glad I added: for `.py` files it pulls out the actual prompt strings instead of counting the whole source file. You care about what the model sees, not your import statements.

A few things it does not do, on purpose. It won't trace a live agent run, won't capture tool responses coming back into context, won't tell you what Claude would charge. That's fine. This is a pre-flight check, not observability. When you want the real per-run picture Langfuse is a solid opensource option. but for my local testing I just want to know if a prompt is bloated before I ship it, this should give you an idea.

Fifty-ish lines. Sometimes that's all a problem needs.

Code's here if it's useful to you: [github.com/AMineer/ollama-token-counter](https://github.com/AMineer/ollama-token-counter)
