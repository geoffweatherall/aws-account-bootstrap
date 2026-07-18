# management-account

CloudFormation for the AWS Organizations **management account** (339140804537).
This deploys the Service Control Policies (SCPs) that lock down which AWS
regions and services member accounts can use - restrictions that member
account admins cannot remove, because SCPs live outside the account they
restrict and are only editable from here.

Only one template: `scp-guardrails.yaml`.

## Prerequisites (should already be true)

- Organizations is enabled on this account, in **"All features"** mode
  (not "Consolidated billing" - SCPs don't work in that mode). Check under
  Organizations console > Settings.
- The member account (431071856068) has already accepted its invitation
  and shows up under Organizations > Accounts.

## What this does

Two SCPs, both attached to the account IDs listed in `pTargetAccountIds`:

- **region-allowlist** - denies any action outside the regions in
  `pAllowedRegions`, except for AWS's inherently-global services (IAM,
  Organizations, Route53, CloudFront, Support, STS, etc.) which are exempt
  so you don't lock yourself out of essentials.
- **service-allowlist** - allows only the service action prefixes in
  `pAllowedServiceActions`. Anything not listed is denied by omission -
  this is what blocks EC2 and everything else you don't use, without
  having to enumerate what to block.

Defaults are scoped to exactly what mootmaker-api/mootmaker-webapp
currently use: `s3, dynamodb, lambda, appsync, cognito-idp, cloudfront,
iam, sts, logs`, plus `budgets, ce, support, trustedadvisor, health` for
account hygiene and a couple of `organizations:Describe*/List*` calls for
basic self-service visibility from the member account.

**Important:** none of this applies to the management account itself - SCPs
never restrict the account they're managed from, only member accounts.
That's intentional; it's what keeps this guardrail out of reach of the
member account's admin user even if their credentials leak.

## Deploying

1. Log in to **339140804537** as the **root user** (console).
2. Go to **CloudFormation** (any region - `us-east-1` is fine;
   `AWS::Organizations::Policy` resources are global regardless of the
   stack's region).
3. **Create stack** > **With new resources** > upload `scp-guardrails.yaml`.
4. Review the parameters - the default `pTargetAccountIds` already
   targets `431071856068`. Adjust `pAllowedRegions` /
   `pAllowedServiceActions` first if you want something different from
   the defaults.
5. No IAM capability checkbox is needed for this template - it only
   creates Organizations policies, not IAM resources.
6. Create the stack.

This takes effect immediately on the target account(s) - unlike joining
the organization (which changes nothing by itself, since the default
`FullAWSAccess` SCP grants everything), attaching these SCPs actively
restricts what the member account can do from that moment on.

## Verifying

From a session **in the member account** (431071856068), confirm the
restrictions are live:

```
# Should fail - ec2 isn't in the allow-list
aws ec2 describe-instances --region us-east-1

# Should fail - region isn't in the allow-list
aws s3 ls --region eu-west-1

# Should still work - s3 is allowed, region is allowed
aws s3api list-buckets --region us-east-1
```

Also worth doing once: a full `terraform apply` / `terraform destroy`
cycle against the `test` environment in mootmaker-api and
mootmaker-webapp, to confirm nothing legitimate got caught by the
service allow-list before you rely on this for real.

## Covering more member accounts, or adjusting the lists later

- **New member account**: add its account ID to `pTargetAccountIds` and
  update the stack. No new SCP required.
- **New service/region needed**: add it to `pAllowedServiceActions` /
  `pAllowedRegions` and update the stack.

Both SCPs together are capped at 5120 characters of policy content each
(the AWS SCP size limit) - comfortably far from that with the current
lists, but worth knowing if the allow-list grows a lot over time.
