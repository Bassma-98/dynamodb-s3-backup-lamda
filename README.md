
# DynamoDB to S3 Scheduled Backup (AWS Lambda + EventBridge Scheduler)

A serverless pipeline built with the AWS CLI and Python. A Lambda function imports 10 users from a public API into DynamoDB. A second Lambda function exports the table to timestamped JSON files in a private S3 bucket every 2 minutes.

- **Region used:** us-east-1
- **Date completed:** 2026-11-26

## 1. Architecture

```text
 https://jsonplaceholder.typicode.com/users
                  |  HTTP GET (10 users)
                  v
 [Lambda #1: import-jsonplaceholder-users]
                  |  boto3 PutItem
                  v
 [DynamoDB: jsonplaceholder-users]  (partition key: id, Number)
                  |  boto3 Scan
                  v
 [Lambda #2: export-dynamodb-to-s3] <--- [EventBridge Scheduler: rate(2 minutes)]
                  |  boto3 PutObject
                  v
 [Private S3 bucket]  backups/YYYY-MM-DD/users-...json

 Both Lambdas write logs and metrics to CloudWatch.
```

**Flow:** Lambda #1 is run manually. It reads the user list from the API and saves each user to DynamoDB. EventBridge Scheduler triggers Lambda #2 every 2 minutes. Lambda #2 scans the whole table and writes a new JSON file to S3, named with the UTC date, time and request ID so nothing is overwritten.

## 2. Languages, tools and services

| Item | Used for | Version / note |
| --- | --- | --- |
| Python | Lambda function code (`lambda_src/`) | Python 3.12 runtime |
| Boto3 | AWS SDK inside Lambda | Included in the Lambda Python runtime |
| Bash | Running the setup commands | Linux, macOS, WSL or Git Bash |
| AWS CLI | Creating and testing all resources | v1 or v2 |
| JSON | IAM policies and trust files | `iam_policies/` |
| Amazon DynamoDB | Stores the users | Provisioned 5 RCU / 5 WCU (free tier friendly) |
| AWS Lambda | Runs the import and export code | 128 MB, 30 s timeout |
| Amazon S3 | Stores the JSON backups | Private, default encryption |
| EventBridge Scheduler | Triggers the export every 2 minutes | `rate(2 minutes)` |
| IAM | Roles and least-privilege policies | One role per Lambda plus one for Scheduler |
| CloudWatch | Logs and metrics | Automatic |

## 3. Setup required before starting

| # | Requirement | How |
| --- | --- | --- |
| 1 | AWS account | Free tier is enough for this lab |
| 2 | IAM user for the CLI | IAM, Users, Create user, attach `AdministratorAccess` for a short lab |
| 3 | Access key for that user | IAM, user, Security credentials, Create access key, choose CLI |
| 4 | AWS CLI installed | `aws --version` |
| 5 | Log in from the terminal | `aws configure` (region `us-east-1`, output `json`) |
| 6 | Unzipped assignment folder | Contains `lambda_zips/` and `iam_policies/` |

**Never commit access keys, the key CSV, or your real account ID.** Delete the access key after finishing.

## 4. Repository structure

```text
.
├── README.md                    <- this report
├── lambda_src/
│   ├── import_users/lambda_function.py
│   └── backup_users/lambda_function.py
├── lambda_zips/                 <- deployable zips
├── iam_policies/                <- policy templates (placeholders only)
├── sample_output/               <- example exported JSON (if kept)
├── screenshots/                 <- evidence
└── .gitignore
```

## 5. Commands used

### A. Variables and login check

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| A1 | `aws --version` | Checks the AWS CLI is installed | Bash, AWS CLI |
| A2 | `aws configure` | Saves access key, secret, region and output format | AWS CLI |
| A3 | `aws sts get-caller-identity` | Confirms the login and shows the account | AWS CLI |
| A4 | `REGION=us-east-1` | Stores the region in a variable | Bash |
| A5 | `ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)` | Stores the 12-digit account ID | Bash, AWS CLI |
| A6 | `BUCKET=student-lambda-backups-$ACCOUNT_ID` | Builds a unique bucket name | Bash |

### B. DynamoDB table

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| B1 | `aws dynamodb create-table --table-name jsonplaceholder-users --attribute-definitions AttributeName=id,AttributeType=N --key-schema AttributeName=id,KeyType=HASH --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5` | Creates the table with `id` (Number) as partition key | AWS CLI |
| B2 | `aws dynamodb wait table-exists --table-name jsonplaceholder-users` | Waits until the table is Active (prints nothing) | AWS CLI |

### C. S3 bucket

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| C1 | `aws s3api create-bucket --bucket $BUCKET --region $REGION` | Creates the bucket (us-east-1 needs no location constraint) | AWS CLI |
| C2 | `aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true` | Keeps the bucket fully private | AWS CLI |

### D. IAM roles and policies

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| D1 | `sed "s/REGION/$REGION/; s/ACCOUNT_ID/$ACCOUNT_ID/" iam_policies/lambda-import-putitem-policy.json > import-policy.json` | Fills the placeholders in a copy of the import policy | Bash, sed |
| D2 | `sed "s/REGION/$REGION/; s/ACCOUNT_ID/$ACCOUNT_ID/; s/YOUR_UNIQUE_BUCKET_NAME/$BUCKET/" iam_policies/lambda-backup-export-policy.json > export-policy.json` | Fills the placeholders in a copy of the export policy | Bash, sed |
| D3 | `cat > trust.json <<EOF ... EOF` | Creates the trust file that lets Lambda use a role (see section 6) | Bash, JSON |
| D4 | `aws iam create-role --role-name import-users-role --assume-role-policy-document file://trust.json` | Creates the role for Lambda #1 | AWS CLI |
| D5 | `aws iam attach-role-policy --role-name import-users-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole` | Lets Lambda #1 write CloudWatch logs | AWS CLI |
| D6 | `aws iam put-role-policy --role-name import-users-role --policy-name AllowPutDemoUsers --policy-document file://import-policy.json` | Gives Lambda #1 only `dynamodb:PutItem` on the table | AWS CLI |
| D7 | `aws iam create-role --role-name export-users-role --assume-role-policy-document file://trust.json` | Creates the role for Lambda #2 | AWS CLI |
| D8 | `aws iam attach-role-policy --role-name export-users-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole` | Lets Lambda #2 write CloudWatch logs | AWS CLI |
| D9 | `aws iam put-role-policy --role-name export-users-role --policy-name AllowDynamoDBJsonBackup --policy-document file://export-policy.json` | Gives Lambda #2 `dynamodb:Scan` and `s3:PutObject` on `backups/*` | AWS CLI |
| D10 | `sleep 15` | Waits for the new roles to become usable | Bash |

### E. Lambda functions

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| E1 | `aws lambda create-function --function-name import-jsonplaceholder-users --runtime python3.12 --handler lambda_function.lambda_handler --role arn:aws:iam::$ACCOUNT_ID:role/import-users-role --zip-file fileb://lambda_zips/lambda-import-users.zip --timeout 30 --memory-size 128 --environment "Variables={TABLE_NAME=jsonplaceholder-users}"` | Creates Lambda #1 from its zip | AWS CLI, Python |
| E2 | `aws lambda create-function --function-name export-dynamodb-to-s3 --runtime python3.12 --handler lambda_function.lambda_handler --role arn:aws:iam::$ACCOUNT_ID:role/export-users-role --zip-file fileb://lambda_zips/lambda-backup-to-s3.zip --timeout 30 --memory-size 128 --environment "Variables={TABLE_NAME=jsonplaceholder-users,BUCKET_NAME=$BUCKET,BACKUP_PREFIX=backups}"` | Creates Lambda #2 with its three environment variables | AWS CLI, Python |

### F. Manual tests

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| F1 | `aws lambda invoke --function-name import-jsonplaceholder-users --cli-binary-format raw-in-base64-out --payload '{}' out1.json` | Runs Lambda #1 once | AWS CLI |
| F2 | `cat out1.json` | Shows the result (`statusCode 200`, `saved_count 10`) | Bash |
| F3 | `aws dynamodb scan --table-name jsonplaceholder-users --select COUNT` | Counts items in the table (expect 10) | AWS CLI |
| F4 | `aws lambda invoke --function-name export-dynamodb-to-s3 --cli-binary-format raw-in-base64-out --payload '{}' out2.json` | Runs Lambda #2 (first export) | AWS CLI |
| F5 | `aws lambda invoke --function-name export-dynamodb-to-s3 --cli-binary-format raw-in-base64-out --payload '{}' out3.json` | Runs Lambda #2 again to prove a second file is created | AWS CLI |
| F6 | `cat out2.json` | Shows `exported_count` and the S3 location | Bash |
| F7 | `aws s3 ls s3://$BUCKET/backups/ --recursive` | Lists the exported JSON files | AWS CLI |

### G. Scheduler (every 2 minutes)

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| G1 | `cat > sched-trust.json <<EOF ... EOF` | Trust file so Scheduler can assume a role (see section 6) | Bash, JSON |
| G2 | `cat > sched-policy.json <<EOF ... EOF` | Policy allowing `lambda:InvokeFunction` on Lambda #2 only | Bash, JSON |
| G3 | `aws iam create-role --role-name scheduler-invoke-export-role --assume-role-policy-document file://sched-trust.json` | Creates the Scheduler execution role | AWS CLI |
| G4 | `aws iam put-role-policy --role-name scheduler-invoke-export-role --policy-name InvokeExport --policy-document file://sched-policy.json` | Attaches the invoke permission | AWS CLI |
| G5 | `cat > target.json <<EOF ... EOF` | Target file with the Lambda ARN, role ARN and input `{}` | Bash, JSON |
| G6 | `aws scheduler create-schedule --name dynamodb-backup-every-2min --schedule-expression "rate(2 minutes)" --flexible-time-window Mode=OFF --target file://target.json` | Creates the schedule | AWS CLI |
| G7 | `aws scheduler get-schedule --name dynamodb-backup-every-2min --query "{State:State,Rate:ScheduleExpression}"` | Checks it is ENABLED with the right rate | AWS CLI |
| G8 | `aws scheduler delete-schedule --name dynamodb-backup-every-2min` | Stops the automatic exports | AWS CLI |
| G9 | `aws scheduler list-schedules --query "Schedules[].Name"` | Confirms the schedule is gone (expect `[]`) | AWS CLI |

### H. Optional: save sample output

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| H1 | `mkdir -p sample_output` | Creates a local folder | Bash |
| H2 | `aws s3 cp s3://$BUCKET/backups/ sample_output/ --recursive --exclude "*" --include "*.json" --only-show-errors` | Downloads the exported JSON files | AWS CLI |

### I. Cleanup

| # | Command | What it does | Language / tool |
| --- | --- | --- | --- |
| I1 | `aws lambda delete-function --function-name import-jsonplaceholder-users` | Deletes Lambda #1 | AWS CLI |
| I2 | `aws lambda delete-function --function-name export-dynamodb-to-s3` | Deletes Lambda #2 | AWS CLI |
| I3 | `aws dynamodb delete-table --table-name jsonplaceholder-users` | Deletes the table | AWS CLI |
| I4 | `aws s3 rm s3://$BUCKET --recursive` | Deletes all files in the bucket | AWS CLI |
| I5 | `aws s3api delete-bucket --bucket $BUCKET` | Deletes the empty bucket | AWS CLI |
| I6 | `aws logs delete-log-group --log-group-name /aws/lambda/import-jsonplaceholder-users` | Deletes Lambda #1 logs | AWS CLI |
| I7 | `aws logs delete-log-group --log-group-name /aws/lambda/export-dynamodb-to-s3` | Deletes Lambda #2 logs | AWS CLI |
| I8 | `aws iam detach-role-policy`, `delete-role-policy`, `delete-role` for the three roles | Removes all lab IAM roles and policies | AWS CLI |
| I9 | Console: IAM, user, Security credentials, delete access key | Removes the CLI key | AWS Console |

## 6. Files created while running

`trust.json` (Lambda trust policy):

```json
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
```

`sched-trust.json` (Scheduler trust policy):

```json
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"scheduler.amazonaws.com"},"Action":"sts:AssumeRole"}]}
```

`sched-policy.json` (allow Scheduler to call Lambda #2):

```json
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"lambda:InvokeFunction","Resource":"arn:aws:lambda:us-east-1:<ACCOUNT_ID>:function:export-dynamodb-to-s3"}]}
```

`target.json` (Scheduler target):

```json
{"Arn":"arn:aws:lambda:us-east-1:<ACCOUNT_ID>:function:export-dynamodb-to-s3","RoleArn":"arn:aws:iam::<ACCOUNT_ID>:role/scheduler-invoke-export-role","Input":"{}"}
```

These generated files are listed in `.gitignore` because they contain the real account ID.



## 7. Report questions

**1. Lambda execution role vs Scheduler execution role.** The Lambda execution role is used by the function's own code to call AWS services (DynamoDB, S3, CloudWatch Logs). The Scheduler execution role is used by EventBridge Scheduler to invoke the Lambda function. They are separate roles with separate permissions.

**2. Why running Lambda #1 twice does not give 20 users.** `PutItem` replaces an existing item that has the same partition key. The same 10 `id` values are overwritten, not added again.

**3. Why each backup has a unique name.** The key contains the UTC date, the time with microseconds, and the Lambda request ID. Every run creates a new object, so earlier backups are never overwritten.

**4. `print()` logs vs Invocations and Errors metrics.** Metrics are automatic numbers: how often the function ran, whether it failed, and how long it took. `print()` output goes to CloudWatch Logs as detailed text. Messages like "Saved user 1" are logs, not metrics.

**5. Table deleted but JSON kept.** The items could be loaded back into a new table. The table settings, indexes, capacity mode, alarms and access policies would not be restored from the JSON alone.

**6. Why an export every 2 minutes is a poor production backup.** A `Scan` is not a point-in-time snapshot. On a large table it uses read capacity and can exceed Lambda time and memory limits. It creates thousands of S3 objects and costs, and it does not save table configuration. Production should use DynamoDB Point-in-Time Recovery and managed Export to S3.

**IAM summary.** Lambda #1 only needs `dynamodb:PutItem` because it only writes users. Lambda #2 needs `dynamodb:Scan` to read the table and `s3:PutObject` to write the backup files. Each role gets only what its function needs (least privilege), not `AdministratorAccess`.

## 9. Notes

- The schedule was deleted after a short run to avoid S3 request costs.
- All lab resources and the CLI access key were removed afterwards.

