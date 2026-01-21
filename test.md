# Building a Self-Service AWS Cleanup Solution for Sandbox Accounts

*I designed and built this end-to-end solution to help platform engineering teams manage AWS sandbox sprawl across multiple development teams. Here's the story of how I created a self-service model that reduced costs by 20% - 30% while empowering developers.*

---

## The Problem: Sandbox Sprawl

If you manage AWS sandbox environments for multiple development teams, you've probably experienced this: developers spin up EC2 instances for quick tests, create RDS databases for prototyping, allocate Elastic IPs for demos—and then forget about them. Days turn into weeks, weeks into months, and suddenly your monthly AWS bill has grown by 40% with no clear understanding of which resources are actually being used.

We faced this exact challenge. With multiple teams using sandbox accounts for experimentation and development, resource sprawl became a significant problem:

- **Cost creep**: Monthly bills growing without corresponding value
- **Resource zombies**: Thousands of forgotten resources running indefinitely
- **Manual burden**: Platform team spending hours investigating and cleaning up
- **Team friction**: Developers frustrated when their resources were accidentally deleted

The traditional approaches weren't working. Full centralized control meant our platform team became a bottleneck, spending time making cleanup decisions they weren't equipped to make. Complete team autonomy meant chaos—no standards, no oversight, and continued cost growth.

This needed something different.

## The Solution: Self-Service with Guardrails

I built a solution around a simple principle: **"Make the right thing the easy thing."**

Our approach combines **centralized infrastructure** with **distributed ownership**. The platform team provides standardized tooling and safety mechanisms, while development teams control their own cleanup policies.

### The Model

```
┌─────────────────────────────────────────────────────────────┐
│         CENTRALIZED                    DISTRIBUTED          │
│         CONTROL                        OWNERSHIP            │
│                                                              │
│  Platform Team Provides:      Teams Control:                │
│  • Infrastructure             • What to protect             │
│  • Workflow orchestration     • Cleanup schedules           │
│  • Security & compliance      • Tag-based filters           │
│  • Approval mechanisms        • Regional scope              │
│  • Monitoring & reporting     • Resource type selection     │
└─────────────────────────────────────────────────────────────┘
```

This hybrid model gives us the best of both worlds:
- Teams get autonomy to manage their own resources
- Platform team maintains standards and oversight
- Finance gets predictable, controlled costs

## Architecture: Three Key Components

The solution consists of three main components, built around [aws-nuke](https://github.com/ekristen/aws-nuke), a powerful open-source tool for removing AWS resources.

### 1. Centralized Infrastructure (Platform Team Manages)

I built a reusable Terraform module that deploys the core infrastructure into each AWS account. This includes:

- **Workflow orchestration** using AWS Step Functions to coordinate the cleanup process
- **Lambda functions** for business logic, dry-run generation, and approval handling
- **CodeBuild projects** that execute aws-nuke with team-specific configurations
- **DynamoDB tables** for state tracking and approval management
- **S3 buckets** for configuration storage and detailed reports
- **SNS topics** for email notifications
- **EventBridge rules** for scheduled execution

The key insight: **deploy once per account, configure many times**. Each team gets the same reliable infrastructure with zero maintenance burden.

### 2. Self-Service Configuration (Teams Manage)

Here's where the magic happens. Teams manage their own cleanup policies through a Git repository with a simple folder structure:

```
configs/
├── team-alpha/
│   └── sandbox-account-123456789012/
│       ├── us-east-1-config.yaml
│       └── us-west-2-config.yaml
└── team-beta/
    └── dev-account-210987654321/
        └── us-east-1-config.yaml
```

The folder naming convention `<account-alias>-<account-id>` is intentional. GitLab CI validates that the account alias in the folder name matches the account ID in the configuration files—a sanity check that ensures configurations are only uploaded to their intended accounts.

Each team defines their cleanup rules using straightforward YAML configuration files. The critical insight: **teams explicitly define what to DELETE using an include-only model, while common protection filters safeguard critical infrastructure**.

Example configuration:

```yaml
regions:
  - us-east-1
  - us-west-2

accounts:
  "123456789012":
    presets:
      # Use common protection filters that all teams copy from
      - "protect-critical-infrastructure"

    # Explicitly define what resource types to DELETE (include-only model)
    resource-types:
      includes:
        - EC2Instance
        - EC2Volume
        - EC2Address
        - RDSInstance
        - LambdaFunction

    filters:
      # Additional protections beyond the common preset
      EC2Instance:
        - property: tag:Environment
          value: "staging"
      EC2Address:
        - property: tag:DoNotDelete
          value: "true"

presets:
  protect-critical-infrastructure:
    filters:
      # Common protections that all teams inherit
      # Protects the aws-nuke infrastructure itself (also covered by IAM permission boundaries)
      IAMRole:
        - "aws-nuke-*"
        - "OrganizationAccountAccessRole"
      VPC:
        - type: "contains"
          value: "default"
      S3Bucket:
        - property: Name
          value: "aws-nuke-*"
      DynamoDBTable:
        - "aws-nuke-*"
      CloudWatchLogsLogGroup:
        - "/aws/codebuild/aws-nuke-*"
        - "/aws/lambda/aws-nuke-*"
```

This configuration approach provides:
- **Include-only deletion**: Teams explicitly list which resource types to clean up (EC2, RDS, Lambda, etc.)
- **Common protection baseline**: All configs inherit standard protections via presets
- **Team-specific filters**: Additional safeguards for resources with specific tags or names
- **Safe by default**: If a resource type isn't in the includes list, it won't be touched

This configuration format follows the [aws-nuke](https://github.com/ekristen/aws-nuke) specification. aws-nuke is a powerful, battle-tested open-source tool that can identify and remove hundreds of AWS resource types across all regions. By building my orchestration layer around aws-nuke, I got:

- ✓ Comprehensive resource coverage (300+ AWS resource types)
- ✓ Active maintenance and community support
- ✓ Proven reliability across thousands of AWS accounts
- ✓ Regular updates for new AWS services

When teams commit changes to their configuration files, GitLab CI automatically validates the syntax and uploads the config to a **staging prefix** in S3. This triggers a dry-run execution, and an approval email is sent to the team with a link to review the dry-run logs. Teams can see exactly what resources will be affected before the configuration goes live. Once approved, the configuration moves from staging to production and becomes active for scheduled executions.

**Pull Request Reviews**: Currently, configuration changes go through a lightweight PR approval process where SRE provides a quick sanity check. This is especially valuable during initial onboarding when teams are learning the patterns. As teams mature and become more familiar with the system, the plan is to remove this dependency and move to fully autonomous configuration management.

### 3. Approval Workflow (Safety Net)

Automation is powerful, but I built in multiple safety layers:

**Layer 1: IAM Permission Boundaries** - Each account has its own custom IAM role with explicit permission boundaries for aws-nuke execution. The role is scoped to that account only and cannot affect resources in other accounts. Even if a team makes a configuration error, the IAM policy prevents deletion of critical infrastructure like VPCs, security groups, network resources, and organizational resources. This is defense-in-depth at the AWS IAM level—the last line of defense against accidental destruction.

**Layer 2: GitLab CI Validation** - The CI pipeline validates that the account alias in the folder name matches the account ID in the configuration, ensuring configs are only uploaded to their intended accounts

**Layer 3: Pull Request Review** - Configuration changes go through PR review where SRE provides guidance during initial onboarding. This is the only point where SRE gets involved, acting as coach rather than gatekeeper. As teams mature, this checkpoint will be removed.

**Layer 4: Configuration Filters** - Teams explicitly define what to protect using YAML configuration

**Layer 5: Dry-Run Previews** - Generate detailed reports before any deletion

**Layer 6: Email Approvals** - Human checkpoint before execution

**Layer 7: Self-Protection** - The system never deletes its own infrastructure

Here's how the workflow operates:

### Initial Configuration or Changes (Approval Required)

When a team creates a new configuration or modifies an existing one:

**Day 1, Config Commit**: Team commits configuration changes to Git
- GitLab CI validates the YAML syntax
- Uploads config to **staging prefix** in S3 (not live yet)
- Triggers dry-run execution with staged config
- Lambda generates preview report from dry-run
- Email sent: "Review 300 resources marked for cleanup - Approve to activate"
- Email includes link to dry-run logs in S3
- Report includes: resource types, IDs, ages, tags, estimated costs

**Day 2, Review & Approval**: Team reviews dry-run logs and approves
- Opens dry-run logs from S3 via email link
- Checks for any resources that should be protected
- If changes needed: Update configuration and restart process
- If satisfied: Approves via email link
- Configuration moves from staging to production S3 prefix
- First cleanup executes on-schedule (EventBridge) after approval

### Scheduled Automatic Execution (No Approval Needed)

Once a configuration is approved, EventBridge runs it automatically on schedule:

**Every Monday, 2:00 AM**: EventBridge triggers the workflow
- Step Function orchestrates the process
- CodeBuild runs aws-nuke with approved configuration
- Resources cleaned up automatically
- Detailed report uploaded to S3
- Team notified of completion

This design reduces operational overhead—teams only need to review and approve when they make configuration changes, not for every scheduled execution.

## Why This Works for Sandbox Accounts

Sandbox environments have unique characteristics that make them perfect for this approach:

✓ **High churn rate** - Resources are created and discarded frequently
✓ **Experimentation focus** - Teams try things and move on
✓ **Cost sensitivity** - Budget constraints demand efficiency
✓ **Lower risk** - Non-production data means more aggressive cleanup is acceptable
✓ **Multiple teams** - Shared responsibility requires clear ownership

The self-service model aligns perfectly with these characteristics. Teams understand their own resources better than anyone else, and they're empowered to make the right decisions.

**Important note**: This approach is specifically designed for sandbox/development accounts. We do **not** recommend it for production environments, shared services, or compliance-heavy workloads where strict retention requirements exist.

## Benefits Across the Organization

### For Development Teams
- ✓ **Autonomy**: Control their own cleanup rules without platform team tickets
- ✓ **Safety**: Protect critical resources with simple tags or filters
- ✓ **Transparency**: See exactly what will be deleted before it happens
- ✓ **Flexibility**: Different rules for different regions or accounts
- ✓ **Speed**: Onboard in 12 minutes, no waiting

### For Platform Team
- ✓ **Standardization**: One solution deployed across all teams
- ✓ **Visibility**: Central reporting and monitoring
- ✓ **Compliance**: Built-in approval workflows and audit trails
- ✓ **Scalability**: New teams self-onboard without platform intervention
- ✓ **Maintenance**: One Terraform module, many deployments

### For Finance/Leadership
- ✓ **Cost Savings**: 40% reduction in sandbox spend
- ✓ **Governance**: Centralized oversight with distributed execution
- ✓ **Metrics**: Track usage patterns and identify waste
- ✓ **Predictability**: Scheduled, consistent cleanup cycles

## Implementation Lessons Learned

### 1. Start with Strong Guardrails

I deployed the infrastructure per account using account-level Terraform variables, which allow for custom IAM roles and permission boundaries when necessary. Each account gets its own IAM role for aws-nuke execution, scoped to operate only within that specific account—it cannot affect resources in other accounts remotely.

The **custom IAM role with permission boundaries** explicitly denies deletion of critical infrastructure components:
- VPCs, subnets, and core networking
- Security groups and NACLs
- AWS Organizations resources
- Critical IAM roles and policies
- Logging and audit infrastructure
- The cleanup system's own resources (the system never deletes itself)

This IAM-level protection means that even if a team completely misconfigures their YAML file (like using empty filters), AWS itself will prevent catastrophic deletions. It's the insurance policy against human error.

Additionally, GitLab CI validates that the account alias in the folder structure (`<account-alias>-<account-id>`) matches the account ID in the configuration files before uploading—ensuring configs can only be deployed to their intended accounts.

### 2. Tag Strategy is Critical

I educated teams on tagging standards and made it part of the onboarding:
- **DoNotDelete**: Universal protection tag
- **Environment**: Distinguish prod-like resources from experiments
- **Owner**: Track resource ownership
- **ExpiresOn**: Optional expiration dates for temporary resources

### 3. Configuration Validation Matters

The GitLab CI pipeline validates every configuration change:
- YAML syntax checking
- Account ID verification
- Region validation
- Filter safety checks (prevent empty filters that would delete everything)

### 4. Observability is Essential

I built comprehensive observability from day one:
- All execution logs in CloudWatch
- All reports archived in S3
- All state tracked in DynamoDB
- Easy to answer: "Who deleted what, when, and why?"

### 5. Progressive Autonomy

Initially, I implemented a lightweight PR approval process for configuration changes. This gave me a chance to:
- Coach teams during their first configurations
- Catch common mistakes early
- Build trust through collaboration

This is the **only touchpoint** where SRE gets involved—no tickets, no lengthy reviews, just a quick sanity check. As teams gain confidence and understanding, the plan is to remove this requirement entirely and achieve fully autonomous configuration management. The goal is to work ourselves out of the approval loop.

## Results

After six months of operation across 15+ teams:

- **20% - 30% reduction** in sandbox AWS spend
- **12 minutes** average onboarding time per team
- **Zero incidents** - IAM permission boundaries and validation working perfectly

The ROI was clear within the first month. More importantly, the conversation shifted from "cost police" to "enablement partner."

## Key Takeaways

If you're considering a similar approach, here are our recommendations:

### 1. Empowerment vs Control
Don't choose between control and autonomy—choose both. The right platform provides guardrails while enabling speed.

### 2. Configuration as Code
If infrastructure is code, cleanup policies should be too. Git gives you version control, peer review, and rollback capabilities.

### 3. Include-Only Deletion Model
Rather than "delete everything except...", use "only delete what you explicitly include". Teams list the specific resource types they want cleaned (EC2, RDS, Lambda) and inherit common protection filters. This safe-by-default approach prevents accidental cleanup of unexpected resource types.

### 4. Safe Automation
Automate aggressively, but fail safely. Multiple layers of protection mean teams can move fast without breaking things.

### 5. Self-Service Scales
Build once, use everywhere. When teams can self-onboard, your solution scales without your team growing.

## Getting Started

If you want to build something similar:

1. **Identify your problem**: Measure your sandbox sprawl and cost impact
2. **Adopt aws-nuke**: Start with [ekristen/aws-nuke](https://github.com/ekristen/aws-nuke) as your foundation
3. **Add orchestration**: Step Functions, Airflow, or your workflow engine to wrap aws-nuke
4. **Build self-service layer**: Configuration repository with CI/CD
5. **Implement approvals**: Email, Slack, or your notification system
6. **Pilot with one team**: Prove the model before scaling
7. **Iterate and improve**: Listen to feedback, refine the experience
8. **Scale organization-wide**: Let success drive adoption

## Conclusion

Platform engineering isn't about gatekeeping—it's about enablement. This self-service AWS cleanup solution embodies this philosophy. I built centralized infrastructure that's reliable and secure, then gave teams the autonomy to configure it for their needs.

The result? Happier developers, lower costs, and a platform engineer who's seen as an enabler rather than a bottleneck.

Sandbox sprawl is a common problem, but the solution doesn't have to be complex. With the right balance of control and autonomy, you can build systems that teams actually want to use.

---

*What challenges are you facing with cloud resource management? How do you balance autonomy and control in your platform? Share your thoughts in the comments.*
