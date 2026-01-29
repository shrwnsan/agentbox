# Adding a New Tool to AgentBox

This guide enables an agent to add support for any agentic coding tool (e.g., GitHub Copilot CLI, Aider, Cursor CLI) to AgentBox.

## Phase 1: Research the Tool

Before implementation, gather this information about the tool:

1. **Installation method**: npm package name, or binary download URL
2. **Command name**: What command runs the tool after installation
3. **Configuration directory**: Where the tool stores config/auth (e.g., `~/.toolname/` or `~/.config/toolname/`)
4. **Authentication**: How the tool authenticates (OAuth, API keys, env vars)
5. **Version command**: How to verify installation (usually `<command> --version`)
6. **Full permissions flag**: Does the tool have a flag to skip permission prompts? (e.g., Claude uses `--dangerously-skip-permissions`). AgentBox runs tools in YOLO mode by default.
7. **Environment variable conflicts**: Does the tool read common env vars (GH_TOKEN, GITHUB_TOKEN, OPENAI_API_KEY, etc.) that may conflict with its own auth? AgentBox loads project `.env` files which may contain tokens for other tools.

**CRITICAL: Do not guess or hallucinate configuration directories.** The config directory must be verified from an authoritative source (official documentation, GitHub repo README, or tool source code). If web search doesn't return the config directory, search more specifically (e.g., "toolname config directory location", "toolname where does it store auth"). If still not found, ask the user to research and provide the config directory.

## Phase 2: Implementation

### Dockerfile

Find the section that installs Claude Code and OpenCode via npm. Add the new package to that install command. If the tool is not an npm package, add a separate installation block.

After installation, add verification that the tool exists and prints its version.

### agentbox script

Make these changes:

**Tool validation**: Find where the script validates the `$tool` variable against allowed values ("claude", "opencode"). Add the new tool name to this check.

**Config directory mount**: Find the `run_container` function where tool-specific mounts are configured (look for the OpenCode and Claude config mount blocks). Add a new `elif` block that:
- Creates the tool's config directory on the host if it doesn't exist
- Mounts it to the same path inside the container
- Logs that the configuration was mounted

Some tools follow XDG conventions and need two mounts:
- `~/.config/toolname/` for configuration
- `~/.local/share/toolname/` for data/auth

**Tool command**: Find where `tool_cmd` is set based on the selected tool. Add an `elif` clause that sets the appropriate command for the new tool. If the tool has a flag to enable full permissions / YOLO mode, include it here (see how Claude adds `--dangerously-skip-permissions`). If the tool has env var conflicts (e.g., reads GH_TOKEN but should use its own OAuth auth), prefix the command with `env -u VAR_NAME` to unset conflicting variables (see how Copilot handles GH_TOKEN).

**Help text**: Update the `show_help` function to:
- List the new tool in the `--tool` option description
- Add a usage example
- Update the environment variable example

### entrypoint.sh

Make these changes:

**PATH configuration**: If the tool installs to a custom bin directory (not `~/.local/bin`), add it to the PATH export at the top of the file.

**Version display**: Find the startup banner section that displays tool versions based on the `$TOOL` environment variable. Add an `elif` clause to display the new tool's version.

### README.md

Update documentation to mention the new tool option.

## Phase 3: User Verification

After completing the implementation, prompt the user with:

---

Run `agentbox --tool <newtool>` to test the new tool.

If it doesn't work, try discarding the changes (`git checkout -- .`) and re-running the prompt. Otherwise, paste the output here and explain your problem.

---

## Notes

- Tool names should be lowercase and simple (e.g., "copilot", "aider")
- Config directories mount from host to persist across container restarts
- Environment variables from `~/.agentbox/.env` and project `.env` are loaded automatically
