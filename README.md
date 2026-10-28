# aws-cfn-guardduty-to-slack

Deploy Amazon GuardDuty across multiple AWS Regions with CloudFormation StackSets, and forward GuardDuty findings (including **AI Protection** findings) to Slack through Amazon EventBridge, SNS, and Amazon Q Developer in chat applications (formerly AWS Chatbot).

[![CI/CD](https://github.com/dceoy/aws-cfn-guardduty-to-slack/actions/workflows/ci.yml/badge.svg)](https://github.com/dceoy/aws-cfn-guardduty-to-slack/actions/workflows/ci.yml)

## What is deployed

- `multi-region-guardduty-stackset.cfn.yml` creates GuardDuty detectors with `AI_PROTECTION: ENABLED` and forwards findings to a regional SNS topic through EventBridge. Updated findings are published every 15 minutes.
- `cross-region-chatbot.cfn.yml` connects the existing regional SNS topics to a Slack channel using Amazon Q Developer in chat applications. The channel role and guardrail policy grant **GuardDuty read-only** access instead of unrestricted AWS administrative command access.
- `iam-roles-for-stacksets.cfn.yml` provisions the standard self-managed CloudFormation StackSets IAM roles.

The configured AWS Regions and SNS topic names remain unchanged, preserving the existing notification ARNs. The StackSet uses the **current account** as its deployment target; it does not configure organization member accounts.

AI Protection monitors CloudTrail data events for Amazon Bedrock, Amazon Bedrock AgentCore, and Amazon SageMaker AI. No additional CloudTrail trail is required. AI Protection is a paid protection plan after its 30-day free trial; check [pricing](https://aws.amazon.com/guardduty/pricing/) and [regional availability](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_regions.html) before deployment. Existing GuardDuty protection plans are not intentionally disabled.

## Deployment

Prerequisites: [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) with configured AWS credentials and permissions to deploy CloudFormation and IAM roles, and a Slack workspace authorized in the [Amazon Q Developer in chat applications console](https://console.aws.amazon.com/chatbot/). Run these commands from the repository root in the AWS account that owns the StackSet. Choose the CloudFormation management Region:

```sh
export AWS_REGION=ap-northeast-1
```

1. Deploy the IAM roles for self-managed StackSets. The template defines named IAM roles, so `CAPABILITY_NAMED_IAM` is required:

   ```sh
   aws cloudformation deploy \
     --region "$AWS_REGION" \
     --template-file iam-roles-for-stacksets.cfn.yml \
     --stack-name iam-roles-for-stacksets \
     --capabilities CAPABILITY_NAMED_IAM
   ```

2. Deploy or update the regional GuardDuty StackSet using its parent CloudFormation stack:

   ```sh
   aws cloudformation deploy \
     --region "$AWS_REGION" \
     --template-file multi-region-guardduty-stackset.cfn.yml \
     --stack-name multi-region-guardduty-stackset
   ```

3. Copy the **Slack workspace ID** and **Slack channel ID** (not the channel name). Replace the placeholders with your real IDs and deploy the notification stack:

   ```sh
   export SLACK_WORKSPACE_ID='YOUR_WORKSPACE_ID'
   export SLACK_CHANNEL_ID='YOUR_CHANNEL_ID'

   aws cloudformation deploy \
     --region "$AWS_REGION" \
     --template-file cross-region-chatbot.cfn.yml \
     --stack-name cross-region-chatbot \
     --capabilities CAPABILITY_IAM \
     --parameter-overrides \
       "SlackWorkspaceId=${SLACK_WORKSPACE_ID}" \
       "SlackChannelId=${SLACK_CHANNEL_ID}"
   ```

   Deploy the notification stack only after the regional SNS topics exist. Do not deploy the same Slack channel configuration in multiple Regions.

4. Verify that GuardDuty AI Protection is enabled in each deployed Region:

   ```sh
   REGION=ap-northeast-1
   DETECTOR_ID=$(aws guardduty list-detectors --region "$REGION" --query 'DetectorIds[0]' --output text)
   aws guardduty get-detector --region "$REGION" --detector-id "$DETECTOR_ID" \
     --query "Features[?Name=='AI_PROTECTION'].[Name,Status]" --output table
   ```

   Confirm that a GuardDuty finding reaches the Slack channel. Verify all StackSet instances are `CURRENT` before assuming AI Protection has been enabled in every Region.

## Operational notes

- The existing StackSet uses **self-managed**, account-local permissions and fixed Region targets. Any additional Region must be added to both the StackSet and the Slack SNS ARN list.
- If GuardDuty is already managed by an AWS Organizations delegated administrator or GuardDuty organizational policies, coordinate feature changes with that administrator; this template is not an organization-wide enrollment policy.
- The self-managed StackSet execution role still uses `AdministratorAccess`. It is a shared bootstrap role: replace it with a dedicated least-privilege execution role **after** auditing other StackSets that may use it, rather than silently restricting and breaking unrelated deployments.
- SNS topics are not encrypted with a customer-managed KMS key. If adding encryption later, the KMS key policy must permit EventBridge to publish; an AWS-managed SNS key alone is not sufficient for this integration.
- The GuardDuty EventBridge rule forwards **all** GuardDuty findings, including AI Protection findings. No severity filter is applied.
