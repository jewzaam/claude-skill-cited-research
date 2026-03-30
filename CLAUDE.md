# Contributor Guidance for cited-research Skill

## Agent Prompt Framing

Do not use expert persona framing in agent prompts (e.g., "You are an expert
in...", "As a senior researcher..."). Use task-oriented framing instead (e.g.,
"You are researching [DIMENSION] for [PROJECT]").

Expert persona prompts reduce factual accuracy by 3.6 percentage points on
MMLU because persona activation competes with factual recall for processing
resources (Hu et al., "PRISM: Expert Personas Improve Alignment but Damage
Accuracy", USC, arXiv:2603.18507, 2026 —
<https://arxiv.org/html/2603.18507v1>). Research agents need maximum factual
recall, not role-playing.

## Research Basis

Design decisions in this skill are documented with supporting evidence in
`references/research-basis.md`. When modifying a behavior that has a
corresponding section in that file, check whether the change is consistent
with the evidence. If new research supersedes an existing finding, update
both the skill instruction and the research basis entry.

## Agent Prompts

Agent prompt templates live in `references/sub-agent-prompts.md`. Keep
prompts lean and task-oriented — longer explicit reasoning instructions
degrade performance in research agents (Xu et al., "Search-R1",
arXiv:2602.19526, 2026). The current prompts are structured for "fast
thinking" (direct search/answer) over "slow thinking" (explicit reasoning
before each action).
