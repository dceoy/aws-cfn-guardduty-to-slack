aws-cfn-guardduty-to-slack
==========================

AWS CloudFormation stacks of Amazon GuardDuty and cross-region notification to Slack

[![Lint](https://github.com/dceoy/aws-cfn-guardduty-to-slack/actions/workflows/lint.yml/badge.svg)](https://github.com/dceoy/aws-cfn-guardduty-to-slack/actions/workflows/lint.yml)

Installation
------------

1.  Check out the repository.

    ```sh
    $ git clone git@github.com:dceoy/aws-cfn-guardduty-to-slack.git
    $ cd aws-cfn-guardduty-to-slack
    ```

2.  Install [Rain](https://github.com/aws-cloudformation/rain) and set `~/.aws/config` and `~/.aws/credentials`.

3.  Create AWS IAM roles to enable use of AWS CloudFormation StackSets.

    ```sh
    $ rain deploy iam-roles-for-stacksets.cfn.yml iam-roles-for-stacksets
    ```

4.  Create a StackSet of Amazon GuardDuty for all available regions using `guardduty-and-sns.cfn.yml`.

5.  Set a Slack client on AWS Chatbot and get the Slack workspace ID.

6.  Deploy a Chatbot stack for GuardDuty.

    ```sh
    $ rain deploy cross-region-chatbot.cfn.yml cross-region-chatbot
    ```
