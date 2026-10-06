# Sprint 2 — Demo 1: Prepare Least-Privilege IAM Permissions

## Purpose

Prepare an ECS task execution role that permits registry authentication, image pulls from one repository and log writing within one log group.

**Learning outcome:** Author a least-privilege IAM policy scoped to the permissions needed for a container deployment.

This demo creates and verifies IAM configuration. Application deployment is covered in Demo 2.

## Before starting

Use **Windows PowerShell**. Docker Desktop must be running with Linux containers, and AWS CLI v2 must be installed.

The supplied demo contains the Compose file, trust policy, permissions policy and permission justification. Learners pull the supplied demo and run its commands locally; no Git commit or push is required. No new project or policy files need to be written. Run the commands below in order; stop if a command reports an error.

| Supplied file | Purpose |
|---|---|
| `docker-compose.yml` | Starts Floci and defines its persistent data location. |
| `trust-policy.json` | Allows ECS tasks to assume the execution role. |
| `execution-policy.json` | Specifies allowed actions and resource scopes. |
| `permission-justification.md` | Records why each permission and wildcard is needed. |

Open PowerShell in the folder containing `docker-compose.yml`. Run policy commands from the folder containing both JSON files. If those files are in a supplied subfolder, open PowerShell in that subfolder when reaching Step 4.

## 1. Start and check Floci

### Start the emulator

```powershell
docker compose up -d
```

**Why:** Starts Floci using the supplied configuration. `-d` keeps it running in the background.

**Expected:** The Floci container starts or is already running. No Compose editing is required.

### Check the container

```powershell
docker compose ps
```

**Why:** Confirms that the Compose service is running.

**Expected:** The Floci service shows a running/Up status and the port mapping for 4566.

### Check the endpoint

```powershell
curl.exe http://localhost:4566/_floci/health
```

**Why:** Confirms that Floci responds to requests, rather than checking only container status. `curl.exe` explicitly uses curl on Windows.

**Expected:** A health response from Floci.

## 2. Check the AWS CLI and local profile

```powershell
aws --version
```

**Why:** Confirms that the AWS CLI is available in this terminal.

**Expected:** Version information beginning with `aws-cli/2`.

```powershell
aws configure list-profiles
```

**Why:** Lists saved profiles so the demo can reuse `floci` if it already exists.

**Expected:** `floci` appears. If present and configured with the values below, continue to Step 3.

### Only if the profile is missing or needs correction

```powershell
aws configure --profile floci
```

**Why:** Saves a named set of local credentials and defaults for the AWS CLI. The name `floci` is referenced by every demo AWS command.

| Prompt | Enter |
|---|---|
| AWS Access Key ID | `test` |
| AWS Secret Access Key | `test` |
| Default region name | `us-east-1` |
| Default output format | `json` |
| Configure AWS skills and the AWS MCP server for your AI coding agent(s), if shown | `n` |

These are dummy credentials for the local emulator. A real AWS account is not required. The optional AI-agent setup is not needed for this demo.

## 3. Confirm the local caller identity

```powershell
aws sts get-caller-identity --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Confirms that the CLI can call Floci using the profile and returns the local account ID.

**Example response:**

```json
{
  "UserId": "000000000000",
  "Account": "000000000000",
  "Arn": "arn:aws:iam::000000000000:root"
}
```

The returned `root` ARN identifies the emulator caller. It is different from the execution role being created.

| Command option | Meaning |
|---|---|
| `--endpoint-url http://localhost:4566` | Sends the request to local Floci. |
| `--profile floci` | Uses the saved local configuration. |
| `--query` | Selects part of a response when used in later commands. |
| `--output text` | Returns a simple text value, such as an account ID or ARN. |

## 4. Review the supplied policies

From the folder containing both JSON files:

```powershell
Get-Content .\trust-policy.json
```

**Why:** Shows which service can assume the role.

**Expected:** The trust policy includes:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Meaning:** ECS tasks may assume this role. This policy does not grant image-pull or log-writing access.

```powershell
Get-Content .\execution-policy.json
```

**Why:** Shows the actions and resources allowed by the permissions policy.

**Expected:** Registry authentication, image-pull permissions and log-writing permissions. Read each statement as **Allow → an operation → on a resource**.

## 5. Put the correct account ID in the permissions policy

### Why the account ID matters

An **ARN — Amazon Resource Name — identifies a resource**. For the repository and logs in this demo, its account ID identifies the account that owns the resource.

For example:

```text
arn:aws:ecr:us-east-1:000000000000:repository/application-service
```

This identifies the `application-service` repository in account `000000000000`, region `us-east-1`.

`ACCOUNT_ID` in a supplied template is a placeholder. It must be replaced with the account ID returned by Floci so the permission applies to the intended resource. A different account ID would point to a different account's resource.

Floci may accept a policy containing the placeholder or allow an operation without enforcing that policy. That does not make the policy correctly scoped. Successful policy creation or application execution is not proof that the resource ARN is correct.

### Read the account ID into a PowerShell variable

```powershell
$FlociAccountId = aws sts get-caller-identity --query Account --output text --endpoint-url http://localhost:4566 --profile floci
if ($LASTEXITCODE -ne 0) { throw 'Could not read the Floci account ID.' }
$FlociAccountId = $FlociAccountId.Trim()
if ($FlociAccountId -notmatch '^\d{12}$') { throw 'Expected a 12-digit account ID.' }
$FlociAccountId
```

**Why:** Obtains the actual local account ID without guessing or manually copying it. The checks stop the demo if the CLI call failed or returned an unexpected value.

**Expected:** A 12-digit account ID, such as `000000000000`. Keep this PowerShell window open so the variable remains available.

### Replace the placeholder in the supplied file

```powershell
$PolicyText = Get-Content .\execution-policy.json -Raw
$PolicyText = $PolicyText.Replace('ACCOUNT_ID', $FlociAccountId)
[System.IO.File]::WriteAllText((Join-Path (Get-Location) 'execution-policy.json'), $PolicyText, [System.Text.UTF8Encoding]::new($false))
```

**Why:** Updates both account ID placeholders automatically. The file is saved as UTF-8 without a byte-order mark for CLI compatibility.

**Expected:** Normally no output. If the supplied file already contains the correct account ID, this command leaves its contents unchanged.

### Verify the resource values

```powershell
$Policy = Get-Content .\execution-policy.json -Raw | ConvertFrom-Json
$Policy.Statement | Select-Object Sid, Resource
$ExpectedRepositoryArn = "arn:aws:ecr:us-east-1:${FlociAccountId}:repository/application-service"
$ExpectedLogArn = "arn:aws:logs:us-east-1:${FlociAccountId}:log-group:/ecs/application-service:log-stream:*"
$ResourceValues = @($Policy.Statement | ForEach-Object { $_.Resource })
if ($ResourceValues -notcontains $ExpectedRepositoryArn -or $ResourceValues -notcontains $ExpectedLogArn) { throw 'The repository or log ARN does not match this demo account. Correct execution-policy.json before continuing.' }
```

**Why:** Confirms both scoped ARNs use the current account and expected resource names. Placeholder replacement alone would not correct an already filled-in but wrong account ID.

**Expected:** Three statement scopes: `*` for registry authentication, the expected repository ARN and the expected log-stream ARN. No error should appear.

The repository and log group do not need to exist to create this policy. Their names describe the intended deployment targets.

## 6. Check JSON syntax and permission justification

```powershell
Get-Content .\trust-policy.json -Raw | ConvertFrom-Json
```

**Why:** Parses the trust policy and catches malformed JSON.

**Expected:** Parsed policy output with no error.

```powershell
Get-Content .\execution-policy.json -Raw | ConvertFrom-Json
```

**Why:** Checks the permissions file after account ID replacement.

**Expected:** Parsed policy output with no error. JSON parsing checks syntax; it does not prove permissions are appropriate.

```powershell
Select-String -Path .\execution-policy.json -Pattern 'ACCOUNT_ID'
```

**Why:** Checks that no account ID placeholder remains.

**Expected:** No output. If a matching line appears, stop and complete Step 5.

```powershell
Get-Content .\permission-justification.md
```

**Why:** Connects every allowed action to a deployment need. Review the following scope and reasons against the supplied policy:

| Action | Scope | Reason |
|---|---|---|
| `ecr:GetAuthorizationToken` | `*` | Obtain registry authentication; this action cannot be restricted to a repository ARN. |
| `ecr:BatchGetImage` | One repository | Retrieve the image manifest required for pulling. |
| `ecr:GetDownloadUrlForLayer` | Same repository | Obtain image-layer download URLs. |
| `ecr:BatchCheckLayerAvailability` | Same repository | Check layers using the standard execution-role pull permissions. |
| `logs:CreateLogStream` | Streams within one log group | Create a stream for container logs. |
| `logs:PutLogEvents` | Same stream scope | Write application activity and errors. |

**Wildcard reasons:** Registry authentication requires `Resource: "*"`. The log-stream wildcard accommodates changing stream names only within `/ecs/application-service`. Neither exception grants all actions.

The execution role does not need image-push, deletion, S3 or administrator permissions. Log group creation is a separate setup operation; it is not granted here.

**Purpose of this file:** permission-justification.md documents why each permission and wildcard is necessary, helping learners and reviewers check least privilege. Floci does not use this file to grant access; the permission rules come from execution-policy.json

## 7. Create the execution role

```powershell
aws iam create-role --role-name application-execution-role --assume-role-policy-document file://trust-policy.json --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Creates a role and associates its trust policy, allowing ECS tasks to assume it. `file://` tells the CLI to read the supplied JSON file.

**Expected:** A role response containing `application-execution-role` and its ARN. For example:

```text
arn:aws:iam::000000000000:role/application-execution-role
```

**If this succeeds, go directly to Step 8.** No extra role-creation commands are required. If the role already exists, use the clearly labelled troubleshooting section below.

## 8. Attach the permissions policy

```powershell
aws iam put-role-policy --role-name application-execution-role --policy-name application-pull-and-logs --policy-document file://execution-policy.json --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Adds the permissions document as an **inline policy**, stored directly on this role. This defines what the role may do after being assumed.

**Expected:** Successful completion, usually with no output. Running it again replaces the inline policy with the same name.

## 9. Verify the role and policies

Run **all four commands** in this section.

### Check the trust policy

```powershell
aws iam get-role --role-name application-execution-role --query Role.AssumeRolePolicyDocument --output json --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Reads back who may assume the stored role.

**Expected:** `ecs-tasks.amazonaws.com` and `sts:AssumeRole`.

### Check the inline policy name

```powershell
aws iam list-role-policies --role-name application-execution-role --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Confirms the permissions policy is actually present on the role.

**Expected:** `application-pull-and-logs`, with no unexpected additional inline policies.

### Check the stored permission rules

```powershell
aws iam get-role-policy --role-name application-execution-role --policy-name application-pull-and-logs --query PolicyDocument --output json --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Confirms the stored actions and resource scopes match the reviewed file, including the correct account ID.

**Expected:** The three statements for registry authentication, repository-scoped image pulls and log writing within the specified log group.

### Check additional managed policies

```powershell
aws iam list-attached-role-policies --role-name application-execution-role --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Checks for additional attached managed policies that could broaden access beyond the inline policy.

**Expected for this demo role:** An empty `AttachedPolicies` list. This is correct: the demo uses an inline policy. Resolve any unexpected policies before describing the overall role as least privilege.

These commands verify stored configuration. They do not demonstrate that Floci rejects unauthorised requests.

## 10. Save the role ARN

Run from the same folder containing the JSON files:

```powershell
$ExecutionRoleArn = aws iam get-role --role-name application-execution-role --query Role.Arn --output text --endpoint-url http://localhost:4566 --profile floci
if ($LASTEXITCODE -ne 0) { throw 'Could not read the execution-role ARN.' }
$ExecutionRoleArn | Set-Content -Path .\execution-role-arn.txt -Encoding utf8
```

**Why:** Records the role identifier for later use. The file is created in the current policy folder.

**Expected:** `execution-role-arn.txt` is created, usually with no output.

```powershell
Get-Content .\execution-role-arn.txt
```

**Why:** Confirms the file contains the expected identifier.

**Expected:** `arn:aws:iam::000000000000:role/application-execution-role` when the local account ID is `000000000000`.

The text file only records the ARN. The actual role and policy are stored in Floci's persistent data. Retain the demo's `data` folder as well as the policy files and saved ARN.

## Keep local files for Demo 2

Floci creates the `data` folder locally when the demo runs. It contains the saved IAM role and policy and is not included in the supplied Git repository.

**Keep the policy files, `execution-role-arn.txt` and Floci's `data` folder on the same computer for Demo 2. Application deployment comes in Demo 2.**

Do not delete the `data` folder after Demo 1.

## Completion checklist

- Floci responds and the `floci` profile works.
- Both resource ARNs use the account ID returned by Floci.
- ECS tasks are trusted to assume `application-execution-role`.
- The inline policy allows only the required actions and resource scopes.
- Every wildcard has a written justification.
- Verification reveals no unexpected additional policies.
- The role ARN and persistent data are retained.


## Optional: Stop Floci after the demo

From the folder containing `docker-compose.yml`:

```powershell
docker compose down
```

**Why:** Stops and removes the Compose container when the demo is finished.

**Expected:** The container stops. The bind-mounted `data` folder remains; do not delete it.

## Troubleshooting — run only when needed

### The `floci` profile cannot be found

Run `aws configure --profile floci` using Step 2's values in the same Windows user account. Profiles are saved per user.

### Floci does not respond

From the Compose folder:

```powershell
docker compose logs floci
```

**Why:** Shows startup/runtime errors that explain a failed endpoint check. Also confirm Docker Desktop is running and Step 1 succeeded.

### The role already exists

First inspect it:

```powershell
aws iam get-role --role-name application-execution-role --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Confirms that the existing role is the intended demo role. If so, align its trust policy:

```powershell
aws iam update-assume-role-policy --role-name application-execution-role --policy-document file://trust-policy.json --endpoint-url http://localhost:4566 --profile floci
```

**Why:** Replaces the existing role's trust policy with the supplied version. Expected: successful completion, usually no output. Continue to Step 8, then perform all verification checks.

### A policy file cannot be found

Check that PowerShell is in the folder containing the supplied JSON files. Filenames must match exactly; `execution-policy.json.txt` is not the expected file.

### A response opens in a pager

Press `q` to return to the prompt. To disable the pager for this PowerShell session:

```powershell
$env:AWS_PAGER = ''
```

**Why:** Makes subsequent CLI responses print directly in the terminal.

## Validation status

The policy examples and command sequence have been reviewed. This revised Windows command sequence has not been executed in the preparation workspace. Complete the expected-result checks during the Windows run.

## References

- [AWS resource ARN format](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html)
- [AWS ECS task execution role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)
- [AWS execution-role permissions](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonECSTaskExecutionRolePolicy.html)
- [AWS ECR action and resource scopes](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonelasticcontainerregistry.html)
