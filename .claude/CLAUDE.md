Unless the requested action is MINIMAL, invoke the sub-agent that best fits the task. If you are unsure which sub-agent to use, ask for clarification. 
If you need to explore the repository, always use a sub-agent.

For new features and bug fixes, it's strongly recommended to follow this flow:
1. `grill-me` skill
2. `write-prd` skill
3. `break-into-tasks` skill
4. `express-implementer` sub-agent
6. `code-review` sub-agent
7. `write-docs` sub-agent
8. `e2e-qa` sub-agent? use Swagger UI via chrome extension or PW
