# 🤝 Organizational Change Management for Security Remediation

## The Reality

> *Security remediation in a running production environment isn't a technical project — it's a change management project.*

## Common Pushback and How to Handle It

### "You're making our job more complicated"

From: Operations engineers, ITSM

Response:

- Acknowledge the inconvenience honestly
- Show the risk in business terms: "If this key leaks, 100+ teams lose their CICD system"
- Demonstrate the new workflow is learnable (provide training, documentation, hands-on support)
- Emphasize: "I'm not taking away your access — I'm giving you BETTER access with your own identity"

### "This isn't our responsibility — talk to cybersecurity"

From: Lead architect, senior engineers

Response:

- Frame it as shared responsibility: "We discovered it, we understand the system, we're best positioned to fix it"
- Escalate to the department director if needed
- Don't get into blame games — focus on the solution

### "Can't we just change the Confluence page?"

From: Anyone looking for a quick fix

Response:

- Removing the page doesn't un-share the key — people already have it
- The key itself is compromised, not just the documentation
- This is an opportunity to fix the root cause, not just the symptom

## Building Your Coalition

| **Stakeholder**      | **Why They Matter**                    | **How to Get Their Support**                        |
|----------------------|----------------------------------------|-----------------------------------------------------|
| Department Director  | Authority to override pushback         | Inform immediately, frame as business risk          |
| ITSM Manager         | Controls team processes and priorities | Honest discussion about risks and timeline          |
| Operations Engineers | Must adopt the new workflow            | Training, documentation, patience, hands-on support |
| Cybersecurity Team   | May need to be informed (compliance)   | Timing is controversial — see below                 |

## The Controversial Question: When to Inform Cybersecurity?

There's no perfect answer:

- Too early: Risk losing control of the remediation. Cybersecurity may mandate actions that disrupt the system or create panic.
- Too late: Risk losing trust. If they discover it independently, your credibility suffers.
- The middle ground: Inform them after you have a plan and initial progress, but before the remediation is complete.

Our approach: We informed the department director immediately and worked with her on the timing of broader communication.

## Communication Strategy

1. Explain WHY — Frame the risk in business terms, not just technical jargon
2. Show the data — CloudTrail logs prove the scope of the exposure
3. Present the plan — Three parallel workstreams with clear timelines
4. Be resilient — Expect pushback at every meeting; don't take it personally
5. Be patient — Organizational change takes time; don't expect overnight buy-in
6. Celebrate wins — When the key is rotated, when the first engineer uses SSO successfully

*License: MIT License*
