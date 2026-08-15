# MCP-Sentinel --- Final Design Freeze

## Threat Model + Security Requirements + Architecture Baseline

**Version:** 2.1 --- Design Freeze\
**Date:** 15 August 2026\
**Status:** Approved baseline for implementation\
**Target:** Python MCP servers using the official MCP Python SDK and
FastMCP\
**Primary objective:** Evidence-based MCP security analysis with
optional controlled runtime verification

------------------------------------------------------------------------

## 1. Design Decision

MCP-Sentinel is formally defined as:

> **An MCP-aware security analysis and controlled verification framework
> for AI-agent tool ecosystems.**

The system discovers MCP tools, models their externally influenced
inputs, traces those inputs through supported code flows, identifies
dangerous security-sensitive operations, profiles the exposed tool
attack surface, and optionally verifies high-confidence findings inside
a hardened local sandbox.

The primary security decision is deterministic and evidence-based. An
LLM is optional and is limited to bounded assistance such as payload
mutation and prioritization.

### Core principle

**MCP context + deterministic data-flow analysis + evidence + safe
verification**

Not:

**LLM + heuristic guessing**

------------------------------------------------------------------------

# 2. Why the Project Is Timely

MCP is now an important interoperability layer for agentic applications.
The official MCP 2026-07-28 specification introduced a stateless
protocol core, routable request headers, cacheable discovery/list
results, authorization hardening, extensions, and updated Tier 1 SDK
support. This makes MCP increasingly relevant to production
infrastructure rather than only experimental local integrations.

NIST's 2026 AI-agent security work states that AI agents introduce
security risks requiring adaptation of established cybersecurity
practices, particularly because agents combine model outputs with
software capabilities and external systems.

OWASP's MCP security work identifies MCP-specific risks including tool
poisoning, supply-chain compromise, command execution/injection, prompt
injection, and authorization weaknesses.

Therefore, MCP-Sentinel is positioned as an **AI-agent security
engineering project**, not merely a generic Python SAST tool.

------------------------------------------------------------------------

# 3. Security Problem

The relevant execution chain is:

``` text
Human / External Content
        |
        v
      AI Agent
        |
        v
     MCP Host
        |
        v
     MCP Client
        |
        v
    MCP Server
        |
        v
       Tool
        |
   +----+----+----------------+
   |         |                |
   v         v                v
 Files     Database        Network
   |         |                |
   +---------+----------------+
             |
             v
       Operating System
```

The security concern is that an AI agent can cause a tool to execute
operations with real-world consequences.

A vulnerable MCP tool can therefore transform an apparently ordinary
agent request into:

-   command execution
-   unauthorized file access
-   internal-network access
-   database compromise
-   unsafe object deserialization
-   data disclosure
-   destructive actions

The first release focuses on vulnerabilities where a source-code
data-flow relationship can be modeled and evaluated reproducibly.

------------------------------------------------------------------------

# 4. Threat Model

## 4.1 Primary attacker

The primary attacker is assumed to influence an input that reaches an
MCP tool.

The attacker may control:

-   direct tool arguments
-   data supplied to the agent that is subsequently passed to a tool
-   content retrieved from external sources
-   malicious test inputs in a security assessment

The first release does not assume that the attacker has direct
operating-system access.

------------------------------------------------------------------------

## 4.2 Trust boundaries

### Boundary A --- Agent to MCP

The agent decides which tool to invoke and which arguments to provide.

### Boundary B --- MCP client to MCP server

The MCP server receives requests that may contain attacker-influenced
arguments.

### Boundary C --- MCP tool to local resources

The tool may access:

-   files
-   processes
-   databases
-   network services
-   credentials
-   environment variables

### Boundary D --- MCP server to external services

Remote tools may access:

-   HTTP endpoints
-   internal services
-   cloud APIs
-   third-party systems

MCP-Sentinel primarily analyzes Boundaries B--D.

------------------------------------------------------------------------

## 4.3 Assets

The project treats the following as security-sensitive:

  Asset               Examples
  ------------------- ---------------------------------------------
  Files               configuration, source code, secrets
  Credentials         API keys, OAuth tokens, environment secrets
  Databases           local and remote database records
  Network             internal services, metadata endpoints
  OS                  shell/process execution
  MCP server          integrity and availability
  Agent authority     capabilities available to the agent
  External services   APIs and cloud resources

------------------------------------------------------------------------

## 4.4 Threat assumptions

The analysis assumes:

1.  Tool arguments may be attacker-controlled unless proven otherwise.
2.  MCP tool implementations may contain ordinary application
    vulnerabilities.
3.  Dangerous APIs are not automatically vulnerable; reachability and
    execution context matter.
4.  Static analysis cannot establish every runtime or authorization
    property.
5.  Dynamic verification must occur only against authorized local
    targets.
6.  The sandbox must be treated as a security boundary.
7.  LLM output is untrusted and cannot by itself establish
    exploitability.

------------------------------------------------------------------------

# 5. Security Scope

## 5.1 Core static vulnerability classes

### CWE-78 --- OS Command Injection

Detect attacker-controlled input reaching dangerous command execution
without adequate neutralization.

### CWE-22 --- Path Traversal

Detect attacker-controlled path data reaching filesystem operations
without sufficient containment or validation.

### CWE-89 --- SQL Injection

Detect attacker-controlled input reaching SQL execution through unsafe
query construction.

### CWE-918 --- Server-Side Request Forgery

Detect attacker-controlled URLs or network destinations reaching
outbound requests without adequate restrictions.

### CWE-502 --- Deserialization of Untrusted Data

Detect attacker-controlled data reaching unsafe deserialization
mechanisms.

These mappings use MITRE CWE as the primary weakness taxonomy.

------------------------------------------------------------------------

# 6. MCP-Specific Security Plane

The project has two analysis planes.

## Plane A --- Code-flow security

``` text
MCP source
    |
    v
Tool parameter
    |
    v
Taint propagation
    |
    v
Dangerous sink
    |
    v
CWE finding
```

## Plane B --- MCP attack-surface security

The scanner additionally profiles:

-   exposed tool capabilities
-   filesystem access
-   command/process capabilities
-   outbound network capabilities
-   database access
-   destructive operations
-   sensitive-data handling indicators
-   tool metadata and schema risk indicators
-   authorization configuration indicators when statically observable
-   suspicious tool-description patterns
-   manifest/configuration changes where available

Plane B findings are reported as **security indicators** unless the
scanner has sufficient evidence to classify them as vulnerabilities.

------------------------------------------------------------------------

# 7. Explicit Non-Goals

The first release does not attempt to:

-   detect every OWASP MCP Top 10 risk
-   prove that an AI model is aligned
-   detect every prompt injection
-   replace a full enterprise IAM system
-   replace CodeQL or Semgrep
-   autonomously attack public systems
-   perform unrestricted exploit generation
-   support every programming language
-   guarantee runtime exploitability from static analysis
-   make unsupported ecosystem-wide coverage claims

------------------------------------------------------------------------

# 8. Security Requirements

## SR-01 --- Safe Input Handling

All source code and configuration analyzed by MCP-Sentinel must be
treated as untrusted input.

The scanner must not execute target source code during static analysis.

------------------------------------------------------------------------

## SR-02 --- MCP Tool Discovery

The system must identify supported MCP tool registrations and construct
a normalized internal representation containing:

-   tool name
-   source file
-   source line
-   parameters
-   parameter types where available
-   registration mechanism
-   capability indicators

------------------------------------------------------------------------

## SR-03 --- Taint Sources

MCP tool parameters must be treated as untrusted sources by default
unless an explicit analysis rule determines otherwise.

------------------------------------------------------------------------

## SR-04 --- Taint Propagation

The engine must support, at minimum:

-   assignments
-   aliases
-   string concatenation
-   f-strings
-   formatted strings
-   function arguments
-   function returns
-   selected container flows
-   branches
-   inter-procedural calls

------------------------------------------------------------------------

## SR-05 --- Scope Awareness

Taint state must be associated with lexical/function scope.

The analyzer must not treat variables with identical names in unrelated
scopes as the same data-flow object.

------------------------------------------------------------------------

## SR-06 --- Sink Recognition

Sink rules must be context-aware.

Each sink rule must specify:

-   callable/API
-   dangerous argument
-   dangerous execution conditions
-   CWE mapping
-   severity baseline
-   confidence modifiers
-   recognized safe alternatives

------------------------------------------------------------------------

## SR-07 --- Sanitizer Recognition

The analyzer must not classify a function as a sanitizer solely because
its name contains terms such as:

`sanitize`, `validate`, `escape`, or `clean`.

Sanitizer rules must be vulnerability-class specific.

------------------------------------------------------------------------

## SR-08 --- Inter-Procedural Analysis

The analyzer must support propagation across selected function
boundaries.

Example:

``` text
MCP parameter
    |
    v
tool()
    |
    v
helper()
    |
    v
builder()
    |
    v
sink()
```

------------------------------------------------------------------------

## SR-09 --- Evidence

Every finding must contain evidence sufficient for a developer to
understand the data flow.

Evidence should include:

-   source
-   propagation path
-   sink
-   relevant source lines
-   sanitizer/validation state
-   CWE
-   severity
-   confidence

------------------------------------------------------------------------

## SR-10 --- Finding State

Every finding must distinguish:

-   severity
-   confidence
-   verification state

These values must never be conflated.

------------------------------------------------------------------------

## SR-11 --- Dynamic Verification

Dynamic verification must only operate against explicitly authorized
local targets.

------------------------------------------------------------------------

## SR-12 --- Sandbox Isolation

Dynamic execution must use a hardened sandbox with, at minimum:

-   network disabled by default
-   read-only root filesystem
-   dropped capabilities
-   no-new-privileges
-   non-root execution
-   CPU limit
-   memory limit
-   PID limit
-   execution timeout
-   deterministic cleanup
-   no host Docker socket
-   no uncontrolled host filesystem mounts

------------------------------------------------------------------------

## SR-13 --- Oracle

The runtime verification layer must use controlled benign canaries or
other deterministic signals.

A payload must not be classified as successful solely because the
process exited without an error.

------------------------------------------------------------------------

## SR-14 --- LLM Isolation

The LLM must not directly control the host environment.

LLM-generated content must pass through:

``` text
LLM
 |
 v
Payload validator
 |
 v
Policy boundary
 |
 v
Sandbox
```

------------------------------------------------------------------------

## SR-15 --- No Public Exploitation

The default system configuration must prevent dynamic testing against
arbitrary public systems.

------------------------------------------------------------------------

## SR-16 --- Reproducibility

Static findings must be reproducible given:

-   source revision
-   rule-set version
-   scanner version
-   configuration
-   analysis mode

Dynamic findings must additionally record:

-   sandbox image/version
-   payload identifier
-   execution configuration
-   oracle result

------------------------------------------------------------------------

## SR-17 --- Auditability

The scanner must provide machine-readable JSON output.

Human-readable HTML output is required for the final MVP.

------------------------------------------------------------------------

# 9. Finding Schema

A canonical finding should resemble:

``` json
{
  "id": "MCP-CMD-001",
  "tool": "execute",
  "source": {
    "parameter": "command",
    "file": "server.py",
    "line": 12
  },
  "flow": [
    {
      "file": "server.py",
      "line": 13,
      "symbol": "command"
    }
  ],
  "sink": {
    "call": "os.system",
    "file": "server.py",
    "line": 14
  },
  "cwe": "CWE-78",
  "severity": "HIGH",
  "confidence": "HIGH",
  "verification": "UNTESTED",
  "sanitization": "NONE_DETECTED",
  "recommendation": "Avoid shell interpretation or use a validated argument array."
}
```

------------------------------------------------------------------------

# 10. Severity and Confidence

These are separate dimensions.

## Severity

Represents potential impact.

Suggested values:

``` text
INFO
LOW
MEDIUM
HIGH
CRITICAL
```

## Confidence

Represents analysis certainty:

``` text
LOW
MEDIUM
HIGH
```

## Verification

Represents runtime evidence:

``` text
UNTESTED
NOT_CONFIRMED
CONFIRMED
BLOCKED_BY_SANDBOX
```

Example:

``` text
Severity: HIGH
Confidence: HIGH
Verification: UNTESTED
```

is valid.

------------------------------------------------------------------------

# 11. Architecture

``` text
+-----------------------------------------------------------+
|                    MCP-SENTINEL CLI                       |
+-------------------------------+---------------------------+
                                |
                                v
+-----------------------------------------------------------+
|                    Repository Ingestion                   |
|  File discovery | Configuration | Language/framework ID  |
+-------------------------------+---------------------------+
                                |
                                v
+-----------------------------------------------------------+
|                    MCP Discovery Layer                    |
|  SDK/FastMCP recognition | Tool registration | Schemas    |
+-------------------------------+---------------------------+
                                |
                                v
+-----------------------------------------------------------+
|                   Static Analysis Core                    |
|                                                           |
| AST -> Symbols -> Scope -> CFG/flow -> Taint -> Findings  |
+-------------------+-------------------+-------------------+
                    |                   |
                    v                   v
              Sink Rules          Sanitizer Rules
                    |                   |
                    +---------+---------+
                              |
                              v
+-----------------------------------------------------------+
|                 Finding & Risk Engine                     |
| CWE | Severity | Confidence | Evidence | Recommendations   |
+-------------------------------+---------------------------+
                                |
                +---------------+---------------+
                |                               |
                v                               v
+---------------------------+       +-----------------------+
| MCP Attack-Surface Model  |       | Dynamic Verification  |
| Tool capabilities         |       | Payloads              |
| Risk indicators           |       | Sandbox               |
| Metadata/schema           |       | MCP client simulator  |
+-------------+-------------+       | Oracle                |
              |                     +-----------+-----------+
              |                                 |
              +---------------+-----------------+
                              |
                              v
+-----------------------------------------------------------+
|                    Reporting Layer                         |
| JSON | HTML | Finding evidence | Risk inventory | Summary |
+-----------------------------------------------------------+

Optional:
+-----------------------------------------------------------+
|              AI Assistance Layer                          |
| Local LLM -> bounded mutation -> validator -> sandbox     |
+-----------------------------------------------------------+
```

------------------------------------------------------------------------

# 12. Internal Modules

``` text
src/mcp_sentinel/
|
├── cli/
├── ingest/
├── mcp/
├── parser/
├── symbols/
├── taint/
├── sinks/
├── sanitizers/
├── findings/
├── scoring/
├── payloads/
├── sandbox/
├── oracle/
├── agent_security/
└── reporting/
```

### Responsibilities

**ingest**\
Repository discovery and source loading.

**mcp**\
MCP SDK/framework detection and tool discovery.

**parser**\
AST and source-location handling.

**symbols**\
Scopes, definitions, references, function signatures.

**taint**\
Source-to-sink propagation.

**sinks**\
Security-sensitive operation rules.

**sanitizers**\
Validation and neutralization semantics.

**findings**\
Canonical security finding model.

**scoring**\
Severity/confidence calculation.

**payloads**\
Deterministic test payload library.

**sandbox**\
Isolated dynamic execution.

**oracle**\
Verification signals.

**agent_security**\
MCP attack-surface and agent-security indicators.

**reporting**\
JSON/HTML reports.

------------------------------------------------------------------------

# 13. Analysis Pipeline

## Stage 1 --- Discover

``` text
Repository
    |
    v
Python project?
    |
    v
MCP implementation?
    |
    v
Tool registrations
```

## Stage 2 --- Normalize

Convert framework-specific registrations into a common model.

## Stage 3 --- Analyze

``` text
Tool parameter
      |
      v
Source
      |
      v
Data-flow graph
      |
      v
Sanitizer checks
      |
      v
Sink checks
```

## Stage 4 --- Score

Produce:

``` text
CWE
Severity
Confidence
Evidence
```

## Stage 5 --- Verify

Only selected high-confidence findings proceed to dynamic testing.

## Stage 6 --- Report

Return:

-   findings
-   attack-surface inventory
-   verification state
-   remediation

------------------------------------------------------------------------

# 14. Dynamic Verification Architecture

``` text
                  STATIC FINDING
                        |
                        v
                Payload Selector
                        |
                        v
                 Policy Validator
                        |
                        v
              +---------------------+
              |   Docker Sandbox    |
              |                     |
              | MCP Server          |
              | MCP Client          |
              | Canary/oracle       |
              +----------+----------+
                         |
                         v
                   Oracle Engine
                         |
              +----------+----------+
              |                     |
              v                     v
          CONFIRMED            NOT_CONFIRMED
```

Dynamic testing is deliberately downstream of static analysis.

This prevents uncontrolled fuzzing of arbitrary code.

------------------------------------------------------------------------

# 15. Attack-Surface Inventory

For every tool:

``` text
Tool
 |
 +-- Inputs
 |
 +-- Filesystem capability
 |
 +-- Process capability
 |
 +-- Network capability
 |
 +-- Database capability
 |
 +-- Destructive capability
 |
 +-- Sensitive-data indicators
 |
 +-- Authorization indicators
 |
 +-- Metadata/schema indicators
 |
 +-- Findings
```

This means MCP-Sentinel remains useful even when it finds no direct CWE
vulnerability.

------------------------------------------------------------------------

# 16. Evaluation Design

## Dataset A --- Vulnerable corpus

Minimum target:

**25+ vulnerable cases**

Cover all five initial CWE classes.

Include:

-   direct flow
-   aliases
-   concatenation
-   f-strings
-   helper functions
-   inter-procedural flows
-   sanitized cases
-   false-positive traps

## Dataset B --- Clean corpus

Minimum target:

**25+ clean cases**

Include legitimate uses of security-sensitive APIs.

## Dataset C --- Real-world corpus

Target:

**30--50 public MCP repositories**

Selection must be documented.

The scanner must not automatically exploit or dynamically test public
targets.

------------------------------------------------------------------------

# 17. Evaluation Metrics

Primary:

-   Recall
-   Precision
-   F1
-   False-positive rate
-   False-negative rate

Secondary:

-   per-CWE recall
-   per-CWE precision
-   inter-procedural improvement
-   sanitizer recognition accuracy
-   median scan time
-   memory use
-   findings per KLOC
-   dynamic confirmation rate

### Engineering gate

The MVP target is:

**Recall ≥ 80% on the controlled vulnerable corpus**

and

**False-positive rate ≤ 30% on the controlled clean corpus**

These are project acceptance thresholds, not claims about universal
real-world performance.

------------------------------------------------------------------------

# 18. Baseline Comparison

Compare against:

-   Bandit
-   Semgrep with documented equivalent rules

The comparison must use:

-   identical test corpus
-   identical expected vulnerability classes
-   documented rule configuration
-   reproducible scanner versions

The experiment asks:

> Does MCP-aware source identification and data-flow modeling add
> measurable detection value?

------------------------------------------------------------------------

# 19. LLM Experiment

The LLM layer is an experiment, not a dependency.

## Control

``` text
Static finding
    |
    v
Deterministic payload
    |
    v
Sandbox
```

## Treatment

``` text
Static finding
    |
    v
Deterministic payload
    |
    v
LLM mutation
    |
    v
Validator
    |
    v
Sandbox
```

Measure:

-   confirmed cases
-   unique confirmed cases
-   payload success rate
-   number of attempts
-   runtime
-   token/model cost
-   invalid payload rate

The LLM is useful only if the experiment demonstrates measurable
benefit.

------------------------------------------------------------------------

# 20. Safety Requirements for the LLM

The LLM:

-   cannot access the host filesystem
-   cannot access the host Docker socket
-   cannot select arbitrary external targets
-   cannot remove sandbox restrictions
-   cannot directly execute commands
-   cannot obtain production credentials
-   cannot bypass payload validation
-   cannot expand the testing scope

All generated content is untrusted.

------------------------------------------------------------------------

# 21. Secure Development Requirements

The project itself must follow secure-development practices.

Required:

-   dependency pinning/lockfile
-   automated dependency auditing
-   secret scanning
-   static linting
-   type checking
-   unit tests
-   integration tests
-   sandbox security tests
-   reproducible builds where practical
-   signed/versioned releases where feasible
-   documented vulnerability disclosure process

------------------------------------------------------------------------

# 22. Ethical and Legal Boundary

MCP-Sentinel is a security research and defensive tool.

Dynamic testing is authorized only for:

-   local test servers
-   intentionally vulnerable lab systems
-   systems for which explicit permission exists

The real-world corpus is for static analysis unless explicit
authorization exists for dynamic testing.

The project must never turn into a system for automated exploitation of
arbitrary public MCP servers.

------------------------------------------------------------------------

# 23. Roadmap After Design Freeze

### Phase 0 --- Foundation

Week 1

-   repository setup
-   threat-model tests
-   MCP version compatibility
-   corpus structure
-   secure CI

### Phase 1 --- Static MVP

Weeks 2--4

-   ingestion
-   MCP discovery
-   AST
-   symbols
-   basic taint
-   sink rules
-   CWE mapping
-   JSON output

### Phase 2 --- Accuracy

Weeks 5--6

-   inter-procedural analysis
-   sanitizer semantics
-   context-aware sinks
-   confidence scoring
-   evaluation gates

### Phase 3 --- MCP Attack Surface

Week 7

-   tool capability profiling
-   metadata/schema indicators
-   authorization indicators
-   risk inventory

### Phase 4 --- Dynamic Verification

Weeks 8--9

-   sandbox
-   client simulator
-   deterministic payloads
-   oracle
-   isolation tests

### Phase 5 --- Evaluation

Week 10

-   baseline comparison
-   ablation studies
-   performance evaluation
-   real-world static corpus

### Phase 6 --- Optional AI

Week 11

-   local LLM
-   bounded mutation
-   control/treatment experiment

### Phase 7 --- Final Release

Week 12

-   reporting
-   documentation
-   demo
-   research report
-   final defense

------------------------------------------------------------------------

# 24. Final Acceptance Criteria

MCP-Sentinel v1 is considered complete only when all of the following
are true:

### Functional

-   MCP Python server detection works.
-   MCP tools are discovered.
-   Tool parameters become taint sources.
-   Supported source-to-sink flows are detected.
-   Inter-procedural flows work for the defined scope.
-   Sanitizer rules work for the defined scope.
-   CWE mapping is deterministic.
-   Findings contain evidence.
-   JSON output is stable.

### Security

-   Static scanning does not execute target code.
-   Dynamic testing is isolated.
-   Network is disabled by default.
-   Sandbox resource limits work.
-   Host filesystem is protected.
-   Dynamic tests cannot reach the host Docker socket.
-   LLM-generated payloads cannot bypass policy controls.

### Research

-   Vulnerable corpus exists.
-   Clean corpus exists.
-   Metrics are reproducible.
-   Baselines are documented.
-   Results are versioned.
-   Limitations are explicitly reported.

### Engineering

-   CLI works from a clean environment.
-   Tests pass in CI.
-   Documentation is sufficient for another student/developer to
    reproduce the evaluation.

------------------------------------------------------------------------

# 25. Design Freeze --- Final Decisions

The following decisions are now considered fixed for implementation
unless evidence requires a documented change:

1.  **Python is the first implementation language.**
2.  **Official MCP Python SDK and FastMCP are the first supported
    frameworks.**
3.  **Deterministic static analysis is the core engine.**
4.  **Taint analysis is the primary detection technique.**
5.  **Sink and sanitizer rules are context-aware.**
6.  **Severity, confidence, and verification are separate fields.**
7.  **Dynamic testing is optional and sandboxed.**
8.  **LLM assistance is optional and bounded.**
9.  **MCP attack-surface profiling is part of the product, but distinct
    from CWE vulnerability detection.**
10. **Evaluation uses controlled ground truth plus carefully selected
    real-world repositories.**
11. **No autonomous public exploitation is permitted.**
12. **The project will not claim unsupported ecosystem-wide coverage or
    novelty.**
13. **The MVP must be useful without the LLM.**
14. **The final system must be explainable enough for academic
    evaluation and developer remediation.**

------------------------------------------------------------------------

# 26. Final Architecture Statement

MCP-Sentinel is therefore defined as a **security analysis pipeline for
the MCP tool boundary**:

``` text
        MCP SERVER
             |
             v
       TOOL DISCOVERY
             |
             v
       INPUT MODELING
             |
             v
       TAINT ANALYSIS
             |
       +-----+------+
       |            |
       v            v
  SANITIZERS      SINKS
       |            |
       +-----+------+
             |
             v
       SECURITY FINDING
             |
       +-----+------+
       |            |
       v            v
 ATTACK-SURFACE   DYNAMIC
    PROFILE       VERIFICATION
                      |
                      v
                  SANDBOX
                      |
                      v
                   ORACLE
                      |
       +--------------+--------------+
       |                             |
       v                             v
   UNCONFIRMED                   CONFIRMED
       |                             |
       +--------------+--------------+
                      |
                      v
              EVIDENCE-BASED REPORT
```

This is the implementation baseline.

------------------------------------------------------------------------

# 27. Reference Basis

The design should be maintained against the following authoritative
sources:

1.  **Model Context Protocol --- 2026-07-28 Specification and release
    documentation.**
2.  **OWASP MCP Top 10.**
3.  **OWASP Agentic AI security guidance.**
4.  **NIST AI-agent security work, including CAISI publications and
    agent identity/authorization work.**
5.  **MITRE CWE for vulnerability classification.**
6.  **Established SAST, taint-analysis, sandboxing, and
    secure-development practices.**

The exact MCP specification revision used for implementation and
evaluation must always be recorded.
