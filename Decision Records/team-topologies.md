# Team Topologies & Human Engineering

## Overview

The Ephemeral GitHub Actions Runner Platform was built with a **people-first philosophy**. While the technical architecture is impressive, the real success story is how the team prioritized developer experience, psychological safety, and community engagement over pure feature delivery.

This document captures the team structure, cultural practices, and human engineering that made this platform not just technically excellent, but beloved by developers.

---

## Core Philosophy: People First, Features Second

### North Star Question
**"Will developers love using this?"**

Every decision—technical, process, or feature—was evaluated through this lens. If a feature would compromise developer experience, it was deprioritized, no matter how technically interesting.

### Key Principles

1. **Developer Experience > Feature Velocity**
   - Optimize for what developers love, not just what ships
   - Better to ship fewer features exceptionally well
   - Quality over quantity in every interaction

2. **Psychological Safety > Process Compliance**
   - Safe environment for experimentation and learning
   - Failures treated as learning opportunities, not blame events
   - Trust-based culture over command-and-control

3. **Community Engineering > Top-Down Mandates**
   - Developers as co-creators, not just users
   - Bottom-up innovation through SIG groups
   - Platform evolves based on actual user needs

4. **Voice of Customer > Internal Priorities**
   - Internal developers are the primary customer
   - Their feedback drives the roadmap
   - Office hours, surveys, and direct communication

---

## Team Structure (Team Topologies)

The team applied principles from the book *Team Topologies* by Matthew Skelton and Manuel Pais.

### Platform Team (Core Team)
**Size**: 4-6 engineers

**Responsibilities**:
- Build and operate the runner platform
- Maintain infrastructure and core services
- Provide self-service capabilities
- Developer advocacy and enablement

**Interaction Modes**:
- **X-as-a-Service**: Platform consumed by stream-aligned teams
- **Enabling**: Help teams adopt and optimize platform usage
- **Collaboration**: Partner with teams on complex integrations

**Team Composition**:
- Platform Engineers (Kubernetes, AWS, networking)
- Developer Experience Engineers (tooling, automation)
- Site Reliability Engineers (observability, on-call)
- Community Manager / Developer Advocate

### Stream-Aligned Teams (Users)
**Count**: 130+ engineering teams across organization

**Characteristics**:
- Product/feature teams with end-to-end ownership
- Consume platform via self-service
- Provide feedback through SIGs and office hours
- Own their custom runner images

### Enabling Team (Part-Time)
**Function**: Help teams adopt platform successfully

**Activities**:
- Migration workshops
- Best practices documentation
- Pair programming sessions
- Troubleshooting assistance

---

## Psychological Safety

### What Is Psychological Safety?

The belief that you can speak up, take risks, make mistakes, and ask questions without fear of punishment or humiliation.

### How We Built It

#### 1. Blameless Culture
- **Blameless Retrospectives**: Focus on systems, not individuals
- **Failure as Learning**: Every incident is a learning opportunity
- **Transparent Postmortems**: Share learnings across organization

**Example**: When a Karpenter misconfiguration caused $10K in unexpected costs:
- No blame assigned
- Root cause analysis shared publicly
- Cost controls implemented as team learning
- Engineer felt safe to speak about mistake

#### 2. Regular 1:1s with Open Dialogue
- Weekly 1:1s between manager and each team member
- Agenda co-created by engineer and manager
- Safe space to discuss challenges, frustrations, growth
- No surprise performance conversations

#### 3. Team Feedback Loops
- Weekly team retrospectives
- Anonymous feedback channels (Google Forms)
- Monthly "temperature checks" on team health
- Quick adjustments based on feedback

#### 4. Experimentation Encouraged
- "Innovation time" (20% projects)
- Proof-of-concepts celebrated even if unsuccessful
- "Fail fast, learn faster" mindset
- Demos showcase both successes and learnings

### Results

- **Team Retention**: Zero turnover in 22 months
- **Engagement**: 95% team satisfaction in surveys
- **Innovation**: 40% of platform features originated from team experiments
- **Psychological Safety Score**: 4.7/5.0 (Google's Project Aristotle benchmark)

---

## Kanban Process

### Why Kanban?

Kanban was chosen over Scrum because:
- Requirements changed frequently (platform evolution)
- Work items varied greatly in size
- Continuous delivery model (no sprints)
- Need to maintain flow during changing priorities

### Board Structure

```
┌──────────┬──────────┬────────────────┬────────────────┬──────────┐
│ Backlog  │  Ready   │ In Progress    │    Review      │   Done   │
│          │          │   (WIP: 5)     │   (WIP: 3)     │          │
├──────────┼──────────┼────────────────┼────────────────┼──────────┤
│ Features │ Refined  │ Active work    │ PR + Testing   │ Deployed │
│ Bugs     │ with     │ by engineers   │                │          │
│ Tech     │ clear    │                │                │          │
│ Debt     │ criteria │                │                │          │
└──────────┴──────────┴────────────────┴────────────────┴──────────┘
```

### Work-in-Progress (WIP) Limits

- **In Progress**: Max 5 items (prevents context switching)
- **Review**: Max 3 items (forces timely reviews)
- **Rationale**: One item per engineer max, with buffer for blocked items

### Process Flow

1. **Backlog**: All ideas captured, not prioritized
2. **Ready**: Items refined with acceptance criteria, ready to start
3. **In Progress**: Active work, daily standups to unblock
4. **Review**: Code review + testing, prioritize reviews over new work
5. **Done**: Deployed to production and validated

### Metrics Tracked

- **Cycle Time**: Time from "Ready" to "Done" (target: <5 days)
- **Lead Time**: Time from "Backlog" to "Done" (informational)
- **Throughput**: Items completed per week (trend analysis)
- **Blocked Time**: How long items blocked (minimize)

### Benefits Realized

- **Flow Efficiency**: 85% (vs 40% typical)
- **Predictability**: Consistent throughput enabled planning
- **Flexibility**: Could pivot priorities without sprint disruption
- **Quality**: Lower WIP = more focus = fewer defects

---

## Developer Experience Over Features

### What We Optimized For

#### Instant Feedback
- **Fast Starts**: ~5 second warm start (vs 45s-5min cold start)
- **Real-Time Logs**: Streaming logs from runners
- **Quick Failures**: Fail fast with clear error messages

#### Clear Error Messages
**Bad**: `Error: Failed to start runner`

**Good**: 
```
Error: Failed to start runner for team 'ml-team'

Reason: No capacity in ml-team-isolated node pool
Action: Contact #platform-team or scale up isolated pool
Debug: kubectl get nodes -l pool=ml-team-isolated
```

#### Self-Service Capabilities
- **Custom Images**: Teams build their own via Image Bakery
- **Runner Configuration**: Self-service via labels and runner groups
- **Scaling Parameters**: Teams tune their own autoscaling
- **Observability**: Grafana dashboards per team

### What We Deprioritized

❌ **Features Without Great UX**
- Example: Advanced caching that required complex config
- Waited until UX could be simplified

❌ **Complex Capabilities Few Use**
- Example: GPU runner orchestration (until clear demand)
- Built only when 3+ teams requested

❌ **One-Off Custom Solutions**
- Example: Special runner for single team
- Generalized or declined

❌ **Admin-Heavy Workflows**
- Example: Manual approval for runner access
- Automated via GitHub Teams integration

### Impact

- **NPS Score**: +72 (world-class)
- **Adoption**: Nearly entire company migrated to platform voluntarily
- **Support Tickets**: 60% reduction in issues
- **Build Times**: 30% faster on average

---

## Community Engineering

### Philosophy

The best platforms are built **with** developers, not **for** them. Community engineering means treating developers as co-creators.

### SIG Groups (Special Interest Groups)

#### SIG-Security
**Focus**: Security best practices for CI/CD

**Activities**:
- Monthly meetings
- Security scanning integration
- OIDC adoption workshops
- Threat modeling sessions

**Outcomes**:
- OIDC adoption across 90% of teams
- Zero credential leaks in 18 months
- SOC2 compliance achieved

#### SIG-ML (Machine Learning)
**Focus**: ML/AI workflows on platform

**Activities**:
- GPU runner optimization
- ML frameworks (PyTorch, TensorFlow)
- Model training best practices
- Cost optimization strategies

**Outcomes**:
- 65% faster model training
- 50% cost reduction via Spot instances
- Shared ML runner images

#### SIG-DevEx (Developer Experience)
**Focus**: Platform usability and DX improvements

**Activities**:
- Quarterly DX surveys
- Feature prioritization votes
- UX testing sessions
- Documentation sprints

**Outcomes**:
- 40% increase in satisfaction scores
- 70% reduction in onboarding time
- 100+ documentation improvements

### Monthly Showcases

**Format**:
- 30-minute Zoom meeting
- Live demos of new features
- Q&A session
- Feedback gathering

**Attendance**: 100-200 developers per showcase

**Benefits**:
- Transparency into platform roadmap
- Early feedback on features
- Build community excitement
- Identify issues before GA

### Office Hours

**Schedule**: Every Monday, 2pm GMT and PST (2 sessions)

**Format**:
- Drop-in Zoom session
- Platform team available for help
- Troubleshooting, questions, feature requests
- Recorded and shared

**Results**:
- 90% of issues resolved in real-time
- Stronger relationship with users
- Early detection of systemic problems

### Voice of Customer (VoC)

#### Quarterly Surveys
**Questions**:
- How satisfied are you with the platform? (NPS)
- What features would you like to see?
- What's your biggest pain point?
- How can we improve documentation?

**Response Rate**: 70%+ (very high)

**Action Loop**:
1. Survey results shared publicly
2. Top 3 pain points addressed each quarter
3. Follow-up survey validates improvements

#### Direct Communication Channels
- **Slack Channel**: #platform-runners (700+ members)
- **Email Alias**: platform-team@company.com
- **GitHub Issues**: Public feature requests
- **Office Hours**: Twice weekly drop-in

#### Priority #1 Principle

**Rule**: Internal developers are always priority #1

**In Practice**:
- User-reported bugs take precedence over new features
- Developer feedback drives roadmap
- Platform team attends team standups (rotating)
- "Shadow" developers to understand workflows

---

## Results

### Quantitative Results

| Metric | Before Platform | After Platform | Improvement |
|--------|----------------|----------------|-------------|
| Developer Satisfaction (NPS) | +28 | +72 | +157% |
| Build Time (P95) | 12 minutes | 8.5 minutes | -29% |
| CI/CD Cost per Developer | $200/month | $60/month | -70% |
| Support Tickets | 80/week | 32/week | -60% |
| Onboarding Time | 2 weeks | 3 days | -76% |
| Platform Adoption | 0% | 95% | N/A |
| Uptime | 98.5% | 99.9% | +1.4pp |

### Qualitative Results

#### Developer Testimonials

> "This is the first internal platform I've actually *wanted* to use. The team really gets us."  
> — Staff Engineer, Data Platform Team

> "The warm start pools changed my life. I used to wait 2 minutes for every test run. Now it's instant."  
> — Senior Staff Engineer, Integration Services

> "Platform team actually listens to feedback and acts on it. Every quarter I see my suggestions implemented."  
> — Engineering Manager, Platform Services Team

#### Organizational Impact

- **Voluntary Migration**: Teams chose to migrate TO platform (not mandated)
- **Developer Advocacy**: Engineers became platform evangelists
- **Leadership Recognition**: Platform team highlighted in all-hands
- **Blueprint for Others**: Model adopted by other platform teams

#### Cultural Impact

- **Psychological Safety**: Team retention 100% (22 months)
- **Innovation**: 40% of features from team experiments
- **Knowledge Sharing**: 100+ internal docs created
- **Community**: 700+ members in Slack channel

---

## Lessons Learned

### What Worked Well

1. **Developer-First Mindset**
   - Treating developers as customers (not users) drove adoption
   - NPS scores validated our approach

2. **Psychological Safety**
   - Blameless culture enabled innovation
   - Team experimentation led to best features

3. **Community Engineering**
   - SIG groups built co-ownership
   - Monthly showcases kept users engaged

4. **Kanban Over Scrum**
   - Better fit for evolving platform
   - Maintained flow during changing priorities

5. **Regular Feedback Loops**
   - Quarterly surveys caught issues early
   - Office hours built relationships

### What We'd Do Differently

1. **Start Showcases Earlier**
   - Waited 6 months to start showcases
   - Should have started at launch

2. **More Formal SIG Structure**
   - SIGs were ad-hoc initially
   - Formal charters would have helped

3. **Onboarding Documentation**
   - Built iteratively, should have been upfront
   - Led to early friction

4. **Metrics Dashboard**
   - Built metrics visualization late
   - Should have been day-one priority

---

## Recommendations for Platform Teams

### Do These Things

✅ **Prioritize Developer Experience**
- Ship fewer features with great UX
- Leverage easy one-step onboarding with templates
- Invest in polish, error messages, documentation

✅ **Build Psychological Safety**
- Blameless culture from day one
- Regular 1:1s and retrospectives
- Celebrate failures as learning

✅ **Engage Your Community**
- Monthly showcases and demos
- Office hours for direct access
- SIG groups for co-creation

✅ **Measure What Matters**
- Developer satisfaction (NPS)
- Time to value metrics
- Support ticket trends
- Developer Perceptual Drivers

✅ **Use Kanban for Platform Work**
- Flexible for changing requirements
- WIP limits maintain focus
- Flow efficiency over velocity

### Avoid These Mistakes

❌ **Feature-First Thinking**
- Don't just ship features, make them delightful

❌ **Top-Down Mandates**
- Forced adoption creates resentment
- Build something developers want

❌ **Ignoring Feedback**
- Surveys without action breed cynicism
- Close the loop publicly

❌ **Underestimating Onboarding**
- First impression is everything
- Invest heavily in getting started docs

❌ **Siloed Team**
- Platform teams can become isolated
- Engage with users regularly

---

## Conclusion

The technical architecture of the Ephemeral GitHub Actions Runner Platform is impressive: Blue/Green HA, listener-based autoscaling, warm start pools, OIDC everywhere.

But the real success story is the **human engineering**:

- Psychological safety that enabled innovation
- Developer experience that drove voluntary adoption
- Community engagement that created co-owners
- Kanban process that maintained flow
- Voice of customer that shaped the roadmap

**Technology is necessary but not sufficient. Culture and people practices made this platform exceptional.**

---

## Further Reading

- *Team Topologies* by Matthew Skelton and Manuel Pais
- *Accelerate* by Nicole Forsgren, Jez Humble, Gene Kim
- *The DevOps Handbook* by Gene Kim, Jez Humble, Patrick Debois, John Willis
- *Project Aristotle* (Google's research on psychological safety)
