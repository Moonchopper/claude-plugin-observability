# claude-plugin-observability (fixture plugin)

> **This is a fixture plugin for the [native-agent-orchestration](https://github.com/Moonchopper/native-agent-orchestration) PoC.**
>
> The plugin is real and installable via `claude plugin install`; the skills it ships are illustrative content for testing the architecture's retrieval and orchestration boundaries against the [`Moonchopper/datadog-operations`](https://github.com/Moonchopper/datadog-operations) fixture repo. It is not intended for production observability work.

## Skills

- `create-log-index` — drives the agent through authoring a Datadog log-index Terraform PR.
- `pr-handoff` — receives a drafted-changes payload and (in the PoC) writes a JSON handoff artifact. A real `gh pr create` implementation is out of scope for the PoC.
