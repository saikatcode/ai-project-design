# Quick Start Guide: Setting Up Your AI Platform Documentation System

---

## Step 1: Set Up Local Repo (Windows Laptop)

```bash
# Create the repo (or clone this downloaded folder)
cd ~/projects
# Copy the ai-platform/ folder here

# Initialize Git
cd ai-platform
git init
git add .
git commit -m "Initial documentation structure"

# (Optional) Push to GitHub/GitLab for mobile access
git remote add origin https://github.com/YOUR_ORG/ai-platform-docs.git
git push -u origin main
```

---

## Step 2: Install and Configure Claude Code

```bash
# Install Claude Code (requires Node.js)
npm install -g @anthropic-ai/claude-code

# Navigate to the repo
cd ~/projects/ai-platform

# Start Claude Code — it will automatically read CLAUDE.md
claude
```

Claude Code reads `CLAUDE.md` on every session start, giving it full context of your project, rules, and conventions.

---

## Step 3: Set Up Claude Projects (for Android/Web Access)

### Project 1: "AI Platform Docs" (Source of Truth)
1. Go to claude.ai → Projects → Create Project
2. Name: "AI Platform Docs"
3. In Project Instructions, paste the contents of `CLAUDE.md`
4. Upload all .md files from `docs/` folder as Project Knowledge
5. Use this project on Android to review, refine, and discuss approved docs

### Project 2: "AI Sandbox" (Exploration)
1. Go to claude.ai → Projects → Create Project
2. Name: "AI Sandbox"
3. In Project Instructions, paste the contents of `SANDBOX-PROJECT-INSTRUCTIONS.md`
4. Do NOT upload any docs — keep it clean for free exploration
5. Use this project on Android or web for brainstorming and testing ideas

---

## Step 4: Daily Workflow

### On Your Laptop (Claude Code) — Heavy Lifting
```bash
cd ~/projects/ai-platform
claude

# Example commands:
# "Read REQ-chatbot-agent-assistant.md and expand ARCH-chatbot-orchestrator-flow.md with component details"
# "I've updated REQ-NBE-003. Cascade changes to the architecture and API contract."
# "Generate the Spring Boot controller for API-nbe-orchestrator-endpoints.md"
# "Add a new risk about embedding model version upgrades"
```

After each session:
```bash
git add .
git commit -m "Updated NBE architecture after requirement cascade"
git push
```

### On Your Phone (Claude Android App) — Review & Explore

**For approved doc work:** Open "AI Platform Docs" project
- "Review the open risks register — are we missing anything for Phase 2?"
- "Summarize the current state of the chatbot API contract"
- "What are all the requirements tagged as Must priority?"

**For exploration:** Open "AI Sandbox" project
- "What if we used event-driven architecture for the voice pipeline instead of sync?"
- "Compare Bedrock Agents vs custom orchestrator — pros and cons for our use case"
- "How would we add A/B testing for different prompt strategies?"

### Promoting from Sandbox to Docs
When a sandbox discussion matures:
1. On Android: Ask Claude "Draft a formal version of our STT evaluation as an ADR"
2. Copy the output
3. On laptop: Create the file via Claude Code or paste into the appropriate .md file
4. Git commit: "Promoted ADR-002 from sandbox discussion"

---

## Step 5: Keeping Projects in Sync

Since Claude Projects (android/web) can't auto-sync with your Git repo, adopt this rhythm:

| Frequency | Action |
|-----------|--------|
| After major Claude Code sessions | Re-upload changed .md files to "AI Platform Docs" project |
| Weekly | Full sync — replace all project knowledge files with latest from repo |
| As needed | Remove outdated files from project to avoid confusion |

**Tip:** Keep a `docs/INDEX.md` file that lists all documents and their current status. Upload this to the project so Claude on Android knows what's current.

---

## Folder Structure Reference

```
ai-platform/
├── CLAUDE.md                                          ← Rules for Claude Code
├── SANDBOX-PROJECT-INSTRUCTIONS.md                    ← Rules for Sandbox project
├── QUICK-START.md                                     ← This file
│
├── docs/                                              ← SPACE 1: Source of Truth
│   ├── requirements/                                  ← Layer 1: WHAT
│   │   ├── REQ-chatbot-agent-assistant.md
│   │   ├── REQ-voice-meeting-assistant.md
│   │   └── REQ-nbe-customer-intelligence.md
│   │
│   ├── architecture/                                  ← Layer 2: HOW
│   │   ├── ARCH-chatbot-orchestrator-flow.md
│   │   ├── ARCH-nbe-pattern3-hybrid.md
│   │   └── ARCH-voice-meeting-flow.md
│   │
│   ├── api-contracts/                                 ← Layer 3: INTERFACE
│   │   ├── API-chatbot-orchestrator-endpoints.md
│   │   ├── API-nbe-orchestrator-endpoints.md
│   │   └── API-voice-orchestrator-endpoints.md
│   │
│   ├── risks-and-decisions/                           ← Layer 4: WHY
│   │   ├── ADR-001-orchestrator-owns-rag-logic.md
│   │   └── RISK-open-risks-register.md
│   │
│   └── test-data/                                     ← Layer 5: VALIDATION
│       ├── TEST-chatbot-complex-query-payloads.md
│       └── TEST-nbe-engagement-plan-scenarios.md
│
└── dev/                                               ← SPACE 2: Code
    └── orchestrator/
        ├── src/
        ├── tests/
        └── README.md
```

---

## Cascade Dependency Map

```
REQ-chatbot-*  ──→  ARCH-chatbot-*  ──→  API-chatbot-*  ──→  TEST-chatbot-*  ──→  dev/orchestrator/chatbot/
REQ-voice-*    ──→  ARCH-voice-*    ──→  API-voice-*    ──→  TEST-voice-*    ──→  dev/orchestrator/voice/
REQ-nbe-*      ──→  ARCH-nbe-*      ──→  API-nbe-*      ──→  TEST-nbe-*      ──→  dev/orchestrator/nbe/

Cross-cutting:
All REQ-*      ──→  RISK-open-risks-register.md
All ARCH-*     ──→  ADR-*.md (decisions that shaped them)
```
