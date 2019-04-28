# About:
https://github.com/awslabs/aws-iam-aad


## Requirements

* AWS CLI already configured with Administrator permission
* [Python 3 installed](https://www.python.org/downloads/)
* [Docker installed](https://www.docker.com/community-edition)

## Setup process

This guide relies on the use of CloudFormation stack set, which is a centralized way of running CloudFormation templates in 
member accounts. First we need to prepare the organizations:
- add the AWSCloudFormationStackSetAdministrationRole and AWSCloudFormationStackSetExecutionRole to the root account 
- and the role AWSCloudFormationStackSetExecutionRole to **all** of the member accounts 

### Prepare the root account

Let's prepare the root account so that to be able to execute Stack Sets:
```bash
aws cloudformation create-stack --stack-name stackset-admin-role \
--capabilities CAPABILITY_NAMED_IAM --profile root-org \
--template-body file://./cfn_templates/root_account_stackset_admin.yaml
```
Check the execution status:
```bash
aws cloudformation describe-stacks --stack-name stackset-admin-role \
--profile root-org --query 'Stacks[0].StackStatus'
```

### Prepare the member accounts

Run the following command once for every member account, using the member credentials (replace the "xxxxyyyyy" with the ID of the root account)
```bash
aws cloudformation create-stack --stack-name cross-account-access-roles \ 
--parameters ParameterKey=TrustedAccountNumber,ParameterValue=xxxxyyyyy \
--capabilities CAPABILITY_NAMED_IAM --profile member1 \
--template-body file://./cfn_templates/cross_account_access_for_invited_members.yaml
```
Wait until the following command returns **"CREATE_COMPLETE"**:
```bash
aws cloudformation describe-stacks --stack-name cross-account-access-roles \ 
--profile member1 --query 'Stacks[0].StackStatus'
```

In case something goes wrong you might want to check the details with:
```bash
aws cloudformation describe-stack-events \
--stack-name cross-account-access-roles \
--profile member1 \
--query 'StackEvents[*].[ResourceType,ResourceStatus,ResourceStatusReason]' \
--output text|column -s $'\t' -t
```

Repeat these steps with all member accounts which were invited (and not created) in your organization.
Once finished, we are ready to use the CloudFormation StackSet to roll out unified configurations in our organization.

### Add IAM Access Role to all accounts using CloudFormation Stacks

At this step we will create a new stack set which will deploy a restricted role, called AWS_IAM_AAD_UpdateTask_CrossAccountRole used by the role synchronizing lambda function in a later step (please not: **"xxxxyyyyy"** is the account number of your root account).

```bash
aws cloudformation create-stack-set --stack-set-name member-iam-access-role --profile root-org \
--parameters ParameterKey=TrustedAccountNumber,ParameterValue=xxxxyyyyy \
--capabilities CAPABILITY_NAMED_IAM \
--template-body file://./cfn_templates/member_iam_access_role.yaml
```
Wait until the following command returns **"Active"** for a stack set name "member-iam-access-role":
```bash
aws cloudformation list-stack-sets --profile root-org
```
or
```bash
aws cloudformation list-stack-sets --profile root-org --query "Summaries[?(@.StackSetName=='member-iam-access-role')].Status | [0]"
```

### Create the IDP providers and a set of initial roles in every organization

Create the necessary roles in all accounts:
```bash
aws cloudformation create-stack-instances --stack-set-name member-iam-access-role \ 
--profile root-org --accounts '["xxxxxxyyyyyy","yyyyzzzzzzz"]' \ 
--regions '["eu-west-1"]' --operation-preferences FailureToleranceCount=0,MaxConcurrentCount=1 \
--query 'OperationId'
```
where "xxxxxxyyyyyy" is the root account, "yyyyzzzzzzz" is one of the member accounts and the list is continued with all member accounts.
Check the execution result (use the ID returned in the previous step as a parameter:
```bash
aws cloudformation describe-stack-set-operation \ 
--stack-set-name member-iam-access-role --profile root-org \ 
--operation-id xxx-yyyy-zzzzzz --query 'StackSetOperation.Status'
```
If you see "RUNNING", wait until it turns into "FAILED" or "SUCCEEDED". If failed, check the status with:
```bash
aws cloudformation list-stack-instances --profile root-org --stack-set-name member-iam-access-role
```
Now let's add a bit of bash-magic and add the SAML metadata to our final CloudFormation template. At this step the SAML provider (eg. Azure Enterprise Application) needs to be set up already.
The metadata.xml is the SAML file downloaded in a previous step from the SAML Provider (Azure in this case).
```bash
sed -e "s/<MetadataDocument>/$(sed 's:/:\\/:g' ./metadata.xml)/" cfn_templates/saml-roles.yaml > cfn_templates/saml-roles-out.yaml
```
>>>
```bash
aws cloudformation create-stack-set --stack-set-name saml-providers-and-roles --profile root-org \
--capabilities CAPABILITY_NAMED_IAM \
--template-body file://./cfn_templates/saml-roles-out.yaml
```
>>>
```bash
aws cloudformation list-stack-sets --profile root-org --query "Summaries[?(@.StackSetName=='saml-providers-and-roles')].Status | [0]"
```
If Active we can install the IDP provider in all accounts.
```bash
aws cloudformation create-stack-instances --stack-set-name saml-providers-and-roles \
--profile root-org --accounts '["xxxxxxyyyyyy","yyyyzzzzzzz"]' \
--regions '["eu-west-1"]' --operation-preferences FailureToleranceCount=0,MaxConcurrentCount=1 \
--query 'OperationId'
```
Check the results (you should see SUCCEEDED):
```bash
aws cloudformation describe-stack-set-operation \
--stack-set-name saml-providers-and-roles --profile root-org \
--operation-id 17d21f5c-1a92-480f-9ac5-37ffeb30e60d --query 'StackSetOperation.Status'
```
## Deploying the role synchronizing lambda function

Let's put the cherry on the cake.

### Local development

**Invoking function locally using a local sample payload**

```bash
sam local invoke HelloWorldFunction --event event.json
```

**Invoking function locally through local API Gateway**

```bash
sam local start-api
```

If the previous command ran successfully you should now be able to hit the following local endpoint to invoke your function `http://localhost:3000/hello`

**SAM CLI** is used to emulate both Lambda and API Gateway locally and uses our `template.yaml` to understand how to bootstrap this environment (runtime, where the source code is, etc.) - The following excerpt is what the CLI will read in order to initialize an API and its routes:

```yaml
...
Events:
    HelloWorld:
        Type: Api # More info about API Event Source: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#api
        Properties:
            Path: /hello
            Method: get
```

## Packaging and deployment

AWS Lambda Python runtime requires a flat folder with all dependencies including the application. SAM will use `CodeUri` property to know where to look up for both application and dependencies:

```yaml
...
    HelloWorldFunction:
        Type: AWS::Serverless::Function
        Properties:
            CodeUri: sync
            ...
```

Firstly, we need a `S3 bucket` where we can upload our Lambda functions packaged as ZIP before we deploy anything - If you don't have a S3 bucket to store code artifacts then this is a good time to create one:

```bash
aws s3 mb s3://BUCKET_NAME
```

Next, run the following command to package our Lambda function to S3:

```bash
sam package \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME
```

Next, the following command will create a Cloudformation Stack and deploy your SAM resources.

```bash
sam deploy \
    --template-file packaged.yaml \
    --stack-name role_sync \
    --capabilities CAPABILITY_IAM
```

> **See [Serverless Application Model (SAM) HOWTO Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-quick-start.html) for more details in how to get started.**

After deployment is complete you can run the following command to retrieve the API Gateway Endpoint URL:

```bash
aws cloudformation describe-stacks \
    --stack-name role_sync \
    --query 'Stacks[].Outputs[?OutputKey==`HelloWorldApi`]' \
    --output table
``` 

## Fetch, tail, and filter Lambda function logs

To simplify troubleshooting, SAM CLI has a command called sam logs. sam logs lets you fetch logs generated by your Lambda function from the command line. In addition to printing the logs on the terminal, this command has several nifty features to help you quickly find the bug.

`NOTE`: This command works for all AWS Lambda functions; not just the ones you deploy using SAM.

```bash
sam logs -n HelloWorldFunction --stack-name role_sync --tail
```

You can find more information and examples about filtering Lambda function logs in the [SAM CLI Documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-logging.html).

## Testing


Next, we install test dependencies and we run `pytest` against our `tests` folder to run our initial unit tests:

```bash
pip install pytest pytest-mock --user
python -m pytest tests/ -v
```

## Cleanup

In order to delete our Serverless Application recently deployed you can use the following AWS CLI Command:

```bash
aws cloudformation delete-stack --stack-name role_sync
```

## Bringing to the next level

Here are a few things you can try to get more acquainted with building serverless applications using SAM:

### Learn how SAM Build can help you with dependencies

* Uncomment lines on `app.py`
* Build the project with ``sam build --use-container``
* Invoke with ``sam local invoke HelloWorldFunction --event event.json``
* Update tests

### Create an additional API resource

* Create a catch all resource (e.g. /hello/{proxy+}) and return the name requested through this new path
* Update tests

### Step-through debugging

* **[Enable step-through debugging docs for supported runtimes]((https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-using-debugging.html))**

Next, you can use AWS Serverless Application Repository to deploy ready to use Apps that go beyond hello world samples and learn how authors developed their applications: [AWS Serverless Application Repository main page](https://aws.amazon.com/serverless/serverlessrepo/)

# Appendix

## Building the project

[AWS Lambda requires a flat folder](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python-how-to-create-deployment-package.html) with the application as well as its dependencies in  deployment package. When you make changes to your source code or dependency manifest,
run the following command to build your project local testing and deployment:

```bash
sam build
```

If your dependencies contain native modules that need to be compiled specifically for the operating system running on AWS Lambda, use this command to build inside a Lambda-like Docker container instead:
```bash
sam build --use-container
```

By default, this command writes built artifacts to `.aws-sam/build` folder.

## SAM and AWS CLI commands

All commands used throughout this document

```bash
# Generate event.json via generate-event command
sam local generate-event apigateway aws-proxy > event.json

# Invoke function locally with event.json as an input
sam local invoke HelloWorldFunction --event event.json

# Run API Gateway locally
sam local start-api

# Create S3 bucket
aws s3 mb s3://BUCKET_NAME

# Package Lambda function defined locally and upload to S3 as an artifact
sam package \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME

# Deploy SAM template as a CloudFormation stack
sam deploy \
    --template-file packaged.yaml \
    --stack-name role_sync \
    --capabilities CAPABILITY_IAM

# Describe Output section of CloudFormation stack previously created
aws cloudformation describe-stacks \
    --stack-name role_sync \
    --query 'Stacks[].Outputs[?OutputKey==`HelloWorldApi`]' \
    --output table

# Tail Lambda function Logs using Logical name defined in SAM Template
sam logs -n HelloWorldFunction --stack-name role_sync --tail
```

