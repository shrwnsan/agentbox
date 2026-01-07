# Security Review - 2026-01

**Reviewer:** Claude Code (GLM 4.7)
**Date:** 2026-01-07
**Scope:** Core AgentBox codebase (agentbox, Dockerfile, entrypoint.sh, GitHub workflows)

---

## Executive Summary

AgentBox demonstrates good security awareness with proper path validation, permission handling, and container isolation. The codebase is designed for YOLO-mode AI agent isolation, which appropriately shapes the threat model. No critical vulnerabilities identified.

**Severity Distribution:**
- Critical: 0
- High: 1
- Medium: 2
- Low: 3

---

## High-Severity Findings

### H-001: Unverified Script Execution in Dockerfile

**Location:** `Dockerfile:48-56, 83-85, 89-95, 115-116, 132`

**Issue:** Multiple third-party scripts are piped directly to shell without hash verification:
- GitHub CLI keyring (line 48)
- uv installer (line 83)
- NVM installer (line 91)
- SDKMAN installer (line 115)
- oh-my-zsh installer (line 132)

**Impact:** Supply chain compromise if any of these distribution channels are compromised.

**Remediation:**
1. Download script to temporary file first
2. Verify checksum against known good value
3. Only execute after verification

**Example:**
```dockerfile
RUN curl -fsSL https://astral.sh/uv/install.sh -o /tmp/install.sh && \
    sha256sum -c <<< "<expected-hash> /tmp/install.sh" && \
    sh /tmp/install.sh && \
    rm /tmp/install.sh
```

**Note:** This increases Dockerfile complexity. May require maintaining checksums in a separate file or updating them with each release.

---

## Medium-Severity Findings

### M-001: GitHub Actions Wildcard Permissions

**Location:** `.github/workflows/claude-code-review.yml:56`

**Issue:** Wildcard permission allows commenting to ANY PR, not just the one being reviewed:
```yaml
claude_args: '--allowed-tools "Bash(gh pr comment:*)"'
```

**Impact:** Compromised workflow token could spam arbitrary PRs with misinformation.

**Remediation:**
Restrict to specific PR number:
```yaml
claude_args: '--allowed-tools "Bash(gh pr comment ${{ github.event.pull_request.number }}:*)"'
```

### M-002: `.env` Secret Exposure Risk

**Location:** `agentbox:378-381`, README documentation

**Issue:** Project recommends storing `GH_TOKEN` in `.env` file at project root, but `.env` is not in `.gitignore`. If committed, secrets are exposed to repository history.

**Impact:** Credential leakage if `.env` is accidentally committed.

**Remediation:**
1. Add `.env` to project `.gitignore` template
2. Add warning in README about `.env` files and git
3. Consider adding pre-commit hook to reject `.env` commits

---

## Low-Severity Findings

### L-001: MCP Auto-Enable Security Implications

**Location:** README, `entrypoint.sh:76-79`

**Issue:** Documentation suggests adding `"enableAllProjectMcpServers": true` to bypass Claude Code bug #6130. This auto-enables all MCP servers without user consent.

**Impact:** MCP servers may provide access to tools users don't expect to be enabled.

**Remediation:**
Document the security implications of this setting so users can make informed decisions.

### L-002: No Container Resource Limits

**Location:** `agentbox:401-410`

**Issue:** Container started without `--cpus`, `--memory`, or `--pids-limit` options.

**Impact:** Runaway AI agent processes could consume host resources (denial-of-service).

**Remediation:**
Consider adding resource limits as optional configuration flags.

### L-003: Sudo Without Password

**Location:** `Dockerfile:75`

**Issue:** Container user configured with `NOPASSWD` sudo access:
```dockerfile
echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME}
```

**Impact:** Privilege escalation if container is compromised (though container isolation mitigates).

**Remediation:**
Document why sudo access is needed. If not required, remove. If required, consider password requirement.

---

## Positive Security Practices

The following practices were observed and commended:

1. **Path Traversal Protection** (`agentbox:138-141`): Explicit `..` detection
2. **System Directory Blocking** (`agentbox:148-156`): Prevents mounting `/bin`, `/etc`, etc.
3. **SSH Permission Hardening** (`entrypoint.sh:29-38`): Sets proper 600/700 permissions
4. **Read-Only Mounts** (`agentbox:275`): `.gitconfig` mounted read-only
5. **Per-Project Isolation**: Dedicated volumes and container names
6. **Ephemeral Containers**: `--rm` flag ensures cleanup
7. **Bash 4+ Requirement**: Modern bash with better security features
8. **Fail-Fast Error Handling**: `set -euo pipefail`

---

## Notes

- Package version pinning is intentionally not done per README (maintenance trade-off)
- SSH key mounting uses dedicated `~/.agentbox/ssh/` directory, which is appropriate isolation
- Project designed for YOLO-mode AI agent execution - threat model reflects this purpose

---

## References

- Claude Code bug #6130: MCP server prompt issue
- Supply chain best practices: Verify before executing
- OWASP Docker security guidelines
