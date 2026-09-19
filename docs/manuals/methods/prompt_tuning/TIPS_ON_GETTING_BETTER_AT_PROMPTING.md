# Basic tips on getting better at prompting

Given how present LLMs are in our work, it's good to have the ability to prompt LLMs well. Here are some tips on how to get better at prompting (from [Mark's talk about LLMs in social science](https://docs.google.com/presentation/d/189VYscba8AbR5PkQg7jU-cuCQItOkuBlAdaOvpBV5sU/edit?slide=id.g3e6f0b6583c_0_1579#slide=id.g3e6f0b6583c_0_1579))

1. Take a [“Prompting 101”](https://www.coursera.org/specializations/prompting-essentials-google) class.
2. Be specific.
3. Less is better.
  - Don’t attach every document or write every single thing.
  - Less + higher quality context > more but lower quality context
4. Add examples.
5. Use [prompt improvers](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-tools).
6. Don’t have super long conversations. Pro tip: Ask an LLM “give me the prompt to give to another LLM to continue our conversation”
7.  Ask questions in ways that the LLMs are trained on:
  - Look up SFT datasets to see what data LLMs are fine-tuned on. The more your queries look like this, the better.
  - Good: markdown, JSON, bulleted list.
  - Bad: free-form text, counting numbers, domain-specific file formats (e.g., .RData), new or less-used software (e.g., LLMs do better on Python than Stata).
8. Track what works!
