# ewelists.com-tools
Repo for main stack templates and documentation for ewelists tools application

**TODO:** bring tools pipeline docs into this page.

## Test Environment Setup

1. **Hosted Zone:** Create Web stack with default SSL certificate.
    ```
    aws cloudformation create-stack --stack-name Web-HostedZone-Tools-Test --template-body file://web-hostedzone.yaml \
      --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **SSL Certificate:** Create SSL certificate. (Get HostedZoneID from outputs of HostedZone stack)
    ```
    aws cloudformation create-stack --region us-east-1 --stack-name Web-SSL-Tools-Test \
    --template-body file://web-sslcert.yaml \
    --parameters ParameterKey=Environment,ParameterValue=test \
      ParameterKey=HostedZone,ParameterValue=ABCDEFGHIJKL123456789

    aws ssm put-parameter --name /tools.ewelists.com/test/SSLCertificateId --type String \
     --value "f38ecd9a-...."
    ```
1. **Web Stack:** Create Web stack with default SSL certificate.
    ```
    aws cloudformation create-stack --stack-name Web-Tools-Test --template-body file://web-infra.yaml \
      --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **Auth Stack:** Create Auth stack
    ```
    aws cloudformation create-stack --stack-name Auth-Tools-Test \
     --template-body file://auth.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --enable-termination-protection \
     --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **Content:** Create a build of the content and copy to S3.  Ensure that cognito related configuration is updated.
    ```
    npm run build
    aws s3 sync build/ s3://test.tools.ewelists.com --delete
    ```
1. **Test:** Browse to https://test.tools.ewelists.com


## Pipeline

Staging & Production:
1. **Hosted Zone:** Create Web stack with default SSL certificate.
    ```
    aws cloudformation create-stack --stack-name Web-HostedZone-Tools-Staging --template-body file://web-hostedzone.yaml \
      --parameters ParameterKey=Environment,ParameterValue=staging
    ```
1. **SSL Certificate:** Create SSL certificate. (Get HostedZoneID from outputs of HostedZone stack)
    ```
    aws cloudformation create-stack --region us-east-1 --stack-name Web-SSL-Tools-Staging \
    --template-body file://web-sslcert.yaml \
    --parameters ParameterKey=Environment,ParameterValue=staging \
      ParameterKey=HostedZone,ParameterValue=ABCDEFGHIJKL123456789

    aws ssm put-parameter --name /tools.ewelists.com/staging/SSLCertificateId --type String \
     --value "f38ecd9a-...."
    ```
1. **Create pipeline:**
    ```
    aws cloudformation update-stack --stack-name Pipeline-Tools  \
      --template-body file://pipeline.yaml  \
      --capabilities CAPABILITY_NAMED_IAM  \
      --parameters ParameterKey=GitHubToken,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github" --with-decryption --query 'Parameter.Value' --output text` \
        ParameterKey=GitHubSecret,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github_secret" --with-decryption --query  'Parameter.Value' --output text`
    ```


Update tools pipeline:
```
aws cloudformation update-stack --stack-name Pipeline-Tools  \
  --template-body file://pipeline.yaml  \
  --capabilities CAPABILITY_NAMED_IAM  \
  --parameters ParameterKey=GitHubToken,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github" --with-decryption --query 'Parameter.Value' --output text` \
    ParameterKey=GitHubSecret,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github_secret" --with-decryption --query  'Parameter.Value' --output text`
```

## Reference
### Admin Create Accounts
```
aws cognito-idp sign-up --region eu-west-1 --client-id YOUR_COGNITO_APP_CLIENT_ID --username admin@example.com --password Passw0rd!

aws cognito-idp admin-confirm-sign-up --region eu-west-1 --user-pool-id YOUR_COGNITO_USER_POOL_ID --username admin@example.com
```
