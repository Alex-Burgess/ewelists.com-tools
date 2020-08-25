# ewelists.com-tools
Repo for main stack templates and documentation for ewelists tools application

**TODO:** bring tools pipeline docs into this page.

## Test Environment Setup

1. **Autg Stack:** Create Auth stack
    ```
    aws cloudformation create-stack --stack-name Auth-Tools-Test \
     --template-body file://auth.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --enable-termination-protection \
     --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **Web Stack:** Create Web stack with default SSL certificate.
    ```
    aws cloudformation create-stack --stack-name Web-Tools-Test --template-body file://web.yaml \
      --parameters ParameterKey=Environment,ParameterValue=test \
      ParameterKey=DefaultSSLCertificate,ParameterValue=true
    ```
1. **Content:** Create a build of the content and copy to S3.
    ```
    npm run build
    aws s3 sync build/ s3://test.tools.ewelists.com --delete
    ```
1. **SSL Certificate:** Create SSL certificate. Stack will remain in CREATE_IN_PROGRESS state until the certificate is validated, so proceed to next step - see [acm docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-certificatemanager-certificate.html) for more details.
    ```
    aws cloudformation create-stack --region us-east-1 --stack-name Web-SSL-Tools-Test \
    --template-body file://web-sslcert.yaml \
    --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **Validate SSL Certificate:** Validate certificate request using console (See [blog](https://aws.amazon.com/blogs/security/easier-certificate-validation-using-dns-with-aws-certificate-manager/) for steps).
1. **Parameter Store:** Create a parameter to store the SSL certificate ID.
    ```
    aws cloudformation describe-stacks --stack-name Web-SSL-Tools-Staging --region us-east-1 \
     --query 'Stacks[].Outputs[?OutputKey==`CertificateArn`].OutputValue' --output text | awk -F\/ '{print $2}'

    aws ssm put-parameter --name /tools.ewelists.com/test/SSLCertificateId --type String \
     --value "f38ecd9a-...."
    ```
1. **Update Web Stack:** Update the main stack to use the SSL certificate.
    ```
    aws cloudformation update-stack --stack-name Web-Tools-Test --template-body file://web.yaml \
      --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **Test:** Browse to https://test.tools.ewelists.com


## Pipeline

First time through the pipeline will fail.  We need to create the

Staging ssl:
aws cloudformation create-stack --region us-east-1 --stack-name Web-SSL-Tools-Staging \
--template-body file://web-sslcert.yaml \
--parameters ParameterKey=Environment,ParameterValue=staging

aws ssm put-parameter --name /tools.ewelists.com/staging/SSLCertificateId --type String \
 --value "f67e99ec-...."

Create pipeline:


Update tools pipeline:
```
aws cloudformation update-stack --stack-name Pipeline-Tools  \
  --template-body file://pipeline.yaml  \
  --capabilities CAPABILITY_NAMED_IAM  \
  --parameters ParameterKey=GitHubToken,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github" --with-decryption --query 'Parameter.Value' --output text` \
    ParameterKey=GitHubSecret,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github_secret" --with-decryption --query  'Parameter.Value' --output text`
```
