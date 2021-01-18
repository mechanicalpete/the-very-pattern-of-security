# The Very Pattern of Security

This repository contains the reference template used in the [The Very Pattern of Security](TODO:) blog post. To run it:

1. Complete the five exports below; and
2. Run the script.

**Remember!** Good cloud hygiene is very important. So always clean-up after yourself and run this when you're ready to delete the stack: `aws --profile "${MY_AWS_PROFILE}" cloudformation delete-stack --stack-name "${MY_STACK_NAME}"`

Pull-requests, questions and comments welcome!

```bash
export MY_AWS_PROFILE=
export MY_STACK_NAME=
export MY_PARAM_BASE_NAME=
export MY_PARAM_ROLE_SESSION_NAME=
export MY_PARAM_EXTERNAL_ID=

# Deploy CloudFormation template:
aws --profile "${MY_AWS_PROFILE}" cloudformation deploy \
    --template-file Template.yml \
    --stack-name "${MY_STACK_NAME}" \
    --capabilities "CAPABILITY_NAMED_IAM" \
    --parameter-overrides \
        "ParamBaseName=${MY_PARAM_BASE_NAME}" \
        "ParamRoleSessionName=${MY_PARAM_ROLE_SESSION_NAME}" \
        "ParamExternalId=${MY_PARAM_EXTERNAL_ID}"

# Display the outputs
aws --profile "${MY_AWS_PROFILE}" cloudformation describe-stacks \
    --stack-name "${MY_STACK_NAME}" \
    --output table \
    --query "Stacks[0].Outputs"

# Grab the outputs
export AWS_ACCESS_KEY_ID=`aws --profile "${MY_AWS_PROFILE}" cloudformation describe-stacks --stack-name "${MY_STACK_NAME}" --output text --query "Stacks[0].Outputs[?OutputKey=='UserCredentialsAccessKey'].OutputValue"`
export AWS_SECRET_ACCESS_KEY=`aws --profile "${MY_AWS_PROFILE}" cloudformation describe-stacks --stack-name "${MY_STACK_NAME}" --output text --query "Stacks[0].Outputs[?OutputKey=='UserCredentialsSecretAccessKey'].OutputValue"`
export AWS_DEFAULT_REGION=ap-southeast-2
export ROLE_ARN=`aws --profile "${MY_AWS_PROFILE}" cloudformation describe-stacks --stack-name "${MY_STACK_NAME}" --output text --query "Stacks[0].Outputs[?OutputKey=='RoleArn'].OutputValue"`

# TESTS - Role assumption

# Test: Valid session name and missing external id
aws sts assume-role --role-arn "${ROLE_ARN}" --role-session-name "${MY_PARAM_ROLE_SESSION_NAME}"
# > An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::111111111111:user/${MY_PARAM_BASE_NAME} is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::111111111111:role/${MY_PARAM_BASE_NAME}Role

# Test: Invalid session name and invalid external id
aws sts assume-role --role-arn "${ROLE_ARN}" --role-session-name "${MY_PARAM_ROLE_SESSION_NAME}_" --external-id  "${MY_PARAM_EXTERNAL_ID}_"
# > An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::111111111111:user/${MY_PARAM_BASE_NAME} is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::111111111111:role/${MY_PARAM_BASE_NAME}Role

# Test: Invalid session name and valid external id
aws sts assume-role --role-arn "${ROLE_ARN}" --role-session-name "${MY_PARAM_ROLE_SESSION_NAME}_" --external-id  "${MY_PARAM_EXTERNAL_ID}"
# > An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::111111111111:user/${MY_PARAM_BASE_NAME} is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::111111111111:role/${MY_PARAM_BASE_NAME}Role

# Test: Valid session name and invalid external id
aws sts assume-role --role-arn "${ROLE_ARN}" --role-session-name "${MY_PARAM_ROLE_SESSION_NAME}" --external-id  "${MY_PARAM_EXTERNAL_ID}_"
# > An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::111111111111:user/${MY_PARAM_BASE_NAME} is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::111111111111:role/${MY_PARAM_BASE_NAME}Role

# Test: Valid session name and valid external id
aws sts assume-role --role-arn "${ROLE_ARN}" --role-session-name "${MY_PARAM_ROLE_SESSION_NAME}" --external-id "${MY_PARAM_EXTERNAL_ID}"

# TESTS - Permission boundaries
export MY_ASSUMED_CREDENTIALS=`aws sts assume-role --role-arn "${ROLE_ARN}" --role-session-name "${MY_PARAM_ROLE_SESSION_NAME}" --external-id  "${MY_PARAM_EXTERNAL_ID}"`
export AWS_ACCESS_KEY_ID=`echo $MY_ASSUMED_CREDENTIALS | jq -r ".Credentials.AccessKeyId"`
export AWS_SECRET_ACCESS_KEY=`echo $MY_ASSUMED_CREDENTIALS | jq -r ".Credentials.SecretAccessKey"`
export AWS_SESSION_TOKEN=`echo $MY_ASSUMED_CREDENTIALS | jq -r ".Credentials.SessionToken"`

# Test: List buckets (allow)
aws s3api list-buckets
# > List of any S3 buckets in the AWS Account

# Test: Create bucket (deny - permission boundary allows, IAM Role policy doesn't)
aws s3api create-bucket --bucket "this-should-fail"
# > An error occurred (IllegalLocationConstraintException) when calling the CreateBucket operation: The unspecified location constraint is incompatible for the region specific endpoint this request was sent to.

# Test: List IAM Roles (deny - permission boundary forbids, role allows)
aws iam list-roles
# > An error occurred (AccessDenied) when calling the ListRoles operation: User: arn:aws:sts::111111111111:assumed-role/${MY_PARAM_BASE_NAME}role/${MY_PARAM_BASE_NAME} is not authorized to perform: iam:ListRoles on resource: arn:aws:iam::111111111111:role/ with an explicit deny
```
