# cache
Yes. I would implement it as a measured optimization project, not as one large change. The main rule is: first measure the current Kiro workflow, then change one cost source at a time so you know which change gives real savings.

Implementation plan

Step	What you implement	Description	Result
1. Create a baseline	Measurement of the current agent	Select 20–30 representative tasks from your normal work. Include code review, test-gap analysis, debugging, and tasks with many tool calls. For each task, record Kiro credits, input tokens, output tokens, cache-read tokens, cache-write tokens, runtime, number of tool calls, and result quality.	You know the current cost per task.
2. Build a usage collector	Small Python measurement script	Store usage after each agent run in JSON or CSV. Use fields such as task_id, credits, input_tokens, cached_read, cached_write, output_tokens, runtime, tool_calls, and success. If Kiro ACP does not expose token details in your version, collect credits through ACP and cache information from /usage during the experiment.	You can compare versions objectively.
3. Audit the current agent context	Context inventory	Run /context show. Identify everything that Kiro receives on each request: agent prompt, steering files, AGENTS.md, resources, MCP tools, Powers, session context, and other instructions. Classify each item as always necessary, sometimes necessary, or rarely necessary.	You find the main sources of repeated tokens.
4. Reduce permanent context	Smaller base agent	Keep only short and essential rules in the permanent agent configuration. Remove duplicated instructions. Do not put large manuals, architecture documentation, or repository documentation directly into persistent resources.	Smaller repeated prompt prefix.
5. Move large instructions to Skills	On-demand Kiro Skills	Convert task-specific instructions into Skills. For example, create separate Skills for code-review, test-gap-analysis, coverage-analysis, and architecture-review. Kiro reads Skill metadata first and loads the full Skill only when it is necessary.	Large instructions are not sent for unrelated tasks.
6. Move large knowledge to Knowledge Base	Retrieval instead of full-context injection	Put project documentation, coding standards, ADRs, large reference documents, and similar content into Kiro Knowledge Base. The agent searches it only when it needs information. Do not use Knowledge Base for very short rules that the agent must always follow.	You stop sending large static documents on every request.
7. Stabilize the prompt prefix	Cache-friendly agent configuration	Keep the beginning of the agent prompt stable. Do not insert timestamps, current working directory, generated IDs, changing file lists, Git hashes, or similar dynamic values into the permanent prompt. Put variable data later in the task input.	Provider prompt caching can reuse a longer prefix.
8. Stabilize tools	Fixed tool set for each agent	Give each custom agent only the tools it needs. Keep tool names, schemas, configuration, and preferably ordering stable. Do not dynamically attach and remove MCP servers during one session unless necessary. This follows the important lesson from OpenAI Codex: changes in MCP tool ordering caused cache misses.	Higher probability of cache hits and less tool metadata.
9. Make conversation history append-only	Cache-friendly session handling	Do not rewrite previous messages when something changes. Add new information as a new message. For example, if the working directory changes, add a new context message instead of changing an earlier system instruction.	Previous prompt prefixes stay identical.
10. Control compaction	Deliberate /compact policy	Do not compact after an arbitrary small number of turns. Compaction reduces context size, but it also changes the conversation prefix and can destroy cache reuse. Start with compaction only when the context becomes large, such as around 70–80% of the usable context window, then measure whether that threshold is good for your workload.	Better balance between context size and cache reuse.
11. Build deterministic analysis scripts	Python instead of LLM reasoning where possible	Move deterministic work out of the agent. Examples: parse coverage files, find changed files, extract functions/classes, calculate metrics, inspect AST/libclang data, filter test results, and summarize structured reports. The LLM should interpret results, not calculate information that Python can calculate exactly.	Lower token use and fewer expensive model operations.
12. Add exact-result caching	Local cache for deterministic tools	Before a script performs expensive analysis, calculate a cache key from its real dependencies. For example: SHA256(source files + coverage report + analyzer version + config). If the key exists, reuse the result. If an input changes, invalidate the result automatically.	Repeated analysis can cost almost nothing.
13. Return small tool outputs	Summary-first tool interface	Do not return a 20,000-line coverage report to Kiro. Store the full result in a file and return a small JSON result such as count, top_findings, affected_files, and report_path. Let the agent read detailed findings only when needed.	Significant reduction of tool-result tokens.
14. Add progressive disclosure	Summary → details → raw data	Design tools so Kiro first receives metadata. Example: 42 uncovered branches found. The agent can then request findings for one class, and only later request the complete branch information. This is usually better than giving the whole dataset immediately.	The model receives only relevant information.
15. Add task-result caching carefully	Cache complete workflow results	For tasks that are fully deterministic with respect to repository state, cache the final analysis result as well. Example: test-gap analysis for commit abc123. Do not reuse cached LLM conclusions when the inputs or instructions changed.	Entire agent runs can sometimes be skipped.
16. Test Auto versus fixed models	Controlled A/B experiment	Kiro Auto can select models based on the task and uses its own cost optimizations. A fixed model can make the environment more stable, but it can also cost more credits. Run your benchmark once with Auto and once with your selected model. Compare credits per successful task, not only tokens.	You know whether model pinning helps your actual workload.
17. Run the optimized benchmark	A/B comparison	Repeat exactly the same baseline tasks. Compare baseline and optimized versions using the same source code and acceptance criteria. Do not optimize only for token count. Compare result quality as well.	Real evidence of savings.
18. Add regression tests	Guard against future cache-breaking changes	Add checks for agent prompt size, number of persistent resources, number of tools, tool schema changes, Skill size, and large tool responses. You can also hash the stable agent configuration and report unexpected changes.	Future changes do not silently destroy your optimization.
19. Create a cost dashboard	Enterprise observability	Aggregate results per agent, task type, repository, and version. Track credits/task, cache-read ratio, input tokens, output tokens, runtime, success rate, and repeated tool executions.	You can see where money is being spent.
20. Optimize based on measurements	Iterative improvement	After the first version, find the largest remaining cost. If tool outputs are still dominant, reduce them. If cache-write tokens are high but cache reads are low, investigate prefix instability. If many agent calls solve trivial tasks, move them to scripts.	Optimization is driven by data instead of assumptions.

The architecture I would target is:

User task
   │
   ▼
Kiro custom agent
   │
   ├── small stable base prompt
   ├── fixed tool definitions
   └── stable configuration
   │
   ▼
Task classification
   │
   ├───────────────┬────────────────┐
   ▼               ▼                ▼
Skill          Knowledge Base   Python tool
on demand      on demand        deterministic
                                   │
                                   ▼
                             Exact cache
                             hash(inputs)
                              │       │
                           HIT       MISS
                            │          │
                            │       run tool
                            │          │
                            └────┬─────┘
                                 ▼
                        Small structured result
                                 │
                                 ▼
                           Kiro reasoning
                                 │
                                 ▼
                              Result
                                 │
                                 ▼
                       Metrics / cost log

For your environment, I would start with Steps 1–3 first, because they tell you where the tokens actually go. Then implement Steps 4–10 as the cache-friendly Kiro layer. After that, Steps 11–15 are probably where you can get the largest additional savings, because your existing workflows already use deterministic Python analysis around C++ tests and coverage, which is a good fit for this architecture.

A practical first milestone would be one optimized agent, for example your test-gap or quality-review workflow. Do not optimize all agents at once. Prove that the first agent reduces credits per successful analysis without reducing developer-rated result quality, then extract the common components into a reusable token-saver Skill and Python library.