# workload-account

CloudFormation for **member/workload accounts** (this account: 431071856068,
and any future ones). Deploy these from *inside* the account being
configured - not from the management account. Two independent templates:

- `admin-guardrail.yaml` - a self-protecting permissions boundary, an
  MFA-gated admin role with a bounded session lifetime, and a group for
  onboarding IAM users onto this pattern.
- `credential-rotation.yaml` - a scheduled Lambda that force-deactivates
  IAM access keys past a configurable age.

They don't depend on each other and can be deployed in either order.

**These are defense in depth, not the primary control.** The real,
admin-proof restriction is the SCPs deployed from the management account
(see `../management-account`). Everything here can, in principle, be
undone by someone with enough IAM permissions in this account - what makes
it worth deploying anyway is that it closes off the easy/obvious paths and
holds even before or independent of the SCPs being attached.

## Deploying

Log in to 431071856068 with your current admin user (or root), open
CloudFormation, **Create stack** > upload the template. Both templates
create named IAM resources, so the console will ask you to acknowledge
**"I acknowledge that AWS CloudFormation might create IAM resources with
custom names"** (`CAPABILITY_NAMED_IAM`) - that's expected.

## admin-guardrail.yaml

Creates:

- **GuardrailPermissionsBoundary** - a managed policy that caps permissions
  to the same region/service allow-list as the management-account SCP,
  and denies modifying or removing itself, and denies creating any new IAM
  user/role that doesn't carry the same boundary (closing the "create an
  unrestricted identity" escape hatch).
- **GuardrailAdminRole** - an assumable role with the boundary attached
  both as its permissions and as its ceiling. Its trust policy only allows
  `sts:AssumeRole` when `aws:MultiFactorAuthPresent` is true, and
  `MaxSessionDuration` caps how long an assumed session can live (default
  4 hours). A bare leaked access key/secret pair, without the matching MFA
  device, cannot assume this role.
- **GuardrailAdmins** - an IAM group. Add a user to it to onboard them:
  they get permission to assume `GuardrailAdminRole` and to self-manage
  their own password/MFA device, and nothing else standing.

### Migrating your existing admin user onto this

Your current IAM user (`geoff.weatherall`) already has broad permissions
attached directly - that's exactly what this pattern replaces. After
deploying the stack:

```
# 1. Add the user to the new group
aws iam add-user-to-group --user-name geoff.weatherall --group-name GuardrailAdmins

# 2. Set up MFA if you haven't already (console: IAM > Users >
#    geoff.weatherall > Security credentials > Assign MFA device).
#    Do this before step 3 - without MFA registered, you won't be
#    able to assume GuardrailAdminRole afterwards.

# 3. Check what's currently attached directly to the user
aws iam list-attached-user-policies --user-name geoff.weatherall
aws iam list-user-policies --user-name geoff.weatherall

# 4. Detach/remove whatever grants broad admin (e.g. if AdministratorAccess
#    is attached directly)
aws iam detach-user-policy --user-name geoff.weatherall \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 5. Optional extra defense in depth: also cap the user itself with the
#    same boundary (get the ARN from the stack's Outputs)
aws iam put-user-permissions-boundary --user-name geoff.weatherall \
  --permissions-boundary <GuardrailBoundaryArn from stack Outputs>
```

After this, `geoff.weatherall`'s standing permissions are just "assume
`GuardrailAdminRole`" and "manage my own password/MFA" - day-to-day
Terraform work happens through a temporary session:

```
aws sts assume-role \
  --role-arn <GuardrailAdminRoleArn from stack Outputs> \
  --role-session-name work \
  --serial-number <your MFA device ARN> \
  --token-code <current MFA code>
```

(or via the console's "Switch Role" feature once signed in with MFA). Use
the returned temporary credentials for CLI/Terraform work; they expire on
their own after `pMaxSessionDurationSeconds`.

### Onboarding a future new user

Create the IAM user, set a password and require MFA enrollment at first
login, then:

```
aws iam add-user-to-group --user-name <new-user> --group-name GuardrailAdmins
```

That's the whole onboarding step - no per-user trust policy edits needed.

### Known caveat: initial MFA enrollment

`EnrollOwnMFA` deliberately doesn't require `aws:MultiFactorAuthPresent`
(a user has no MFA session before they've enrolled a device - chicken and
egg). That means someone who obtains only a user's password, before that
user has ever set up MFA, could in principle enroll their own MFA device
first. Mitigate this by enrolling MFA immediately when a user is created,
before handing out the password for regular use, rather than leaving MFA
setup for "later."

## credential-rotation.yaml

Runs `iam-key-rotation-enforcer` (Lambda, Python) once a day by default,
deactivating (not deleting) any Active IAM access key older than
`pMaxKeyAgeDays` (default 90). Deactivation is reversible by an admin if
it was a mistake, but ordinary users have no standing permission to
reactivate their own keys - that's deliberate, otherwise the enforcement
would be optional.

If you set `pNotificationEmail`, you'll get an email whenever a key is
deactivated (the subscription needs confirming once, via a link AWS emails
you after deploy). You can also add more subscribers to the
`RotationNotificationTopic` output ARN later without redeploying.

This applies account-wide to every IAM user's access keys, including any
permanent key you keep around for CLI convenience to call `sts assume-role`
from outside a browser. Worth minimizing use of that key regardless -
day-to-day work through `GuardrailAdminRole`'s temporary sessions is safer,
since those expire on their own without needing this Lambda to catch them.

## Keeping the allow-lists in sync

`pAllowedRegions` / `pAllowedServiceActions` here should match the SCP in
`../management-account/scp-guardrails.yaml`. The SCP is the authoritative,
tamper-proof ceiling regardless of what's set here; this template's copy
is a second layer that also works before the SCP is attached, or for an
account not yet covered by it. If you add a new AWS service to the SCP,
add it here too and update this stack.
