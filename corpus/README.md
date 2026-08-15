# Corpus

- `vulnerable/` — controlled test cases with known, injected vulnerabilities (one per CWE class)
- `clean/` — controlled test cases that use security-sensitive APIs safely
- `real_world/` — real public MCP server repositories, used for static analysis only.
  Each entry must record: source URL, commit hash, license, and date collected.
  No dynamic testing is permitted against anything in this directory.