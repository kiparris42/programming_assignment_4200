# AI Use Disclosure — PA1

_Generative AI is allowed on PA1 as a tutor, reviewer, debugging partner and
helper with tooling. It is not allowed to be the author of your work. See the PA1
brief, "AI, Authorship and You". Fill in every section. "None" is an acceptable
answer only if it is true._

## 1. Did you use generative AI on this assignment?

- [ ] No. I did not use any generative AI tool for any part of PA1.
- [x] Yes, as described below.

## 2. Statement (the syllabus form)

> The author acknowledges the utilization of ChatGPT, Google (Gemini), and Claude, a language model
> developed by OpenAI, Google, and Anthropic in the preparation of this assignment. ChatGPT, Google (Gemini), and Claude
> we're employed in the following manner(s) within this assignment: In the recalling of the 
  python syntax. In the running of Wireshark on the command line. And the formatting of analysis responses. And bugfixing the server. And explaining the server instructions easier.

## 3. Log

_One row per substantive interaction. Summaries are fine; you do not need to paste
whole conversations, but keep them in case you are asked._

| # | What I asked (summary) | What I used from the answer | What I changed or rejected | How I checked it was right |
|---|---|---|---|---|
| 1 | Help recalling Python syntax. | Explanations of indentation, file reading/writing, binary encoding, and SHA-256. | Corrected indentation, adjusted variable names, adapted code examples to match what I needed. | Ran Python compilation and the instructor's self-check, which passed all execution tests (30/30). |
| 2 | Capturing/analyzing network traffic. | Using tcpdump and tshark, and checking TCP TIME-WAIT. | Corrected directory paths and used a 200-byte capture limit instead of the initial unlimited capture setting. | Verified the packet capture, compared uploaded/downloaded files, and inspected the actual TCP packets. |
| 3 | Help formatting and organizing my packet analysis. | Markdown formatting, tables, and explanations based on measurements we collected from my capture. | Fixed misplaced sections, removed duplicate information, and corrected formatting issues. | Compared the written values against tshark output and verified the eight required capture facts. |
| 4 | Help with explaining the ETP protocol. | What order to try to work on and idea of isolating each response type into seperate functions | Removed excess imported libraries. | Checked against the self-check execution tests. |
| 5 | Bugfixing and checking the server. | Added most of the try except blocks. | Removed excess imported libraries. | Checked against the self-check execution tests. |

## 4. Where AI-produced material remains

_List each file and function (or document section) that still contains code or
text an AI tool generated, even after you edited it. If none, say none._

src/client.py — run(args): Contains code taken and modified from Google's AI-enhanced web search along with traditional web sources such as 
https://realpython.com/python-sockets/
https://docs.python.org/3.15/
etc

All functions written in server.py were bugfixed by Claude.

evidence/ANALYSIS.md: ChatGPT helped format explanations using measurements collected from our actual packet capture.

## 5. One thing the AI got wrong — or how you checked it

_Describe one answer, suggestion or piece of code that was wrong, misleading or
did not fit the specification, and how you caught it. If you found nothing wrong,
say so and describe how you checked (a test you ran, the part of the
specification you compared it with). If you did not use AI, describe one thing
you got wrong yourself and how you caught it._

ChatGPT initially suggested using `tcpdump -s 0` to capture network traffic, but the assignment required a 200-byte snapshot length. We caught this by checking the assignment instructions, changed the command to use `-s 200`, and verified that the resulting packet capture reported a snapshot length of 200 bytes.