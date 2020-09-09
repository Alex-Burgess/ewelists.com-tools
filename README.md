# ewelists.com-tools
Repo for the main stack templates and documentation for ewelists tools application.

- **Dashboard and Web Tools:** A dashboard type application with a number of aditional tools to Create and Update product details, e.g. prices, etc.  [Url](https://tools.ewelists.com)
- **Notfound Check:** A function which runs on a schedule to check if there are any items in the Notfound table which need our attention (i.e. images and price updating).
- **Backups:** A function that creates backups of the DynamoDB tables.

Some of the tools make use a [product scraping API](https://www.scrapinghub.com/) with [login](https://app.scrapinghub.com/account/login/).


## Contents

- [Overview](#overview)
- [Test Environment Deployment](#test-environment-deployment)
- [Staging and Production Deployment](#staging-and-production-deployment)
- [Useful Information](#reference)

## Overview
The Tools application is based on the same Serverless architecture pattern as the main application.  The application consists of:

- **Web application:** This is a React web application, based on [Material Dashboard React (Free)](https://demos.creative-tim.com/material-dashboard-react/#/admin/dashboard), which provides a template.
- **Authentication:** We use Cognito to provide basic username and password authentication to the web application.
- **Services:** This is a microservice API architecture, using API Gateway and Lambda (python).  AWS SAM provides the framework for the project structure, builds, packaging and deployment.
- **Database:** The functions typically interact with the existing application DynamoDB tables.
- **Pipeline:** A CI/CD pipeline automates the deployments to the staging and production environments.

## Test Environment Deployment
The steps below are used to build the test environment of the application.

1. **Auth Stack:**
    ```
    aws cloudformation create-stack --stack-name Auth-Tools-Test \
     --template-body file://auth.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --enable-termination-protection \
     --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **DNS Hosted Zone:**
    ```
    aws cloudformation create-stack --stack-name Web-HostedZone-Tools-Test --template-body file://web-hostedzone.yaml \
      --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **SSL Certificate:** (Get HostedZoneID from outputs of HostedZone stack, must be in us-east-1 region for use with CloudFront.)
    ```
    aws cloudformation create-stack --region us-east-1 --stack-name Web-SSL-Tools-Test \
    --template-body file://web-sslcert.yaml \
    --parameters ParameterKey=Environment,ParameterValue=test \
      ParameterKey=HostedZone,ParameterValue=ABCDEFGHIJKL123456789

    aws ssm put-parameter --name /tools.ewelists.com/test/SSLCertificateId --type String \
     --value "f38ecd9a-...."
    ```
1. **Web Stack:**
    ```
    aws cloudformation create-stack --stack-name Web-Tools-Test --template-body file://web-infra.yaml \
      --parameters ParameterKey=Environment,ParameterValue=test
    ```
1. **Services Secrets:**
    ```
    aws ssm put-parameter --name /Tools/ScrapingHub/APIKey --type SecureString \
     --value "f38ecd9a-...."
    ```
1. **Services Stack:**
    ```
    aws s3 mb s3://sam-builds-tools-test

    sam build

    sam package \
        --output-template-file packaged.yaml \
        --s3-bucket sam-builds-tools-test

    sam deploy \
        --template-file packaged.yaml \
        --stack-name Service-Tools-test \
        --capabilities CAPABILITY_NAMED_IAM
    ```
1. **Build and Deploy Content:** (Ensure that Cognito and API configuration is updated in config.js)
    ```
    npm run build
    aws s3 sync build/ s3://test.tools.ewelists.com --delete
    ```
1. **Create Test User:**
    ```
    aws cognito-idp sign-up --region eu-west-1 --client-id YOUR_COGNITO_APP_CLIENT_ID --username admin@example.com --password Passw0rd!
    aws cognito-idp admin-confirm-sign-up --region eu-west-1 --user-pool-id YOUR_COGNITO_USER_POOL_ID --username admin@example.com
    ```
1. **Test:** Browse to https://test.tools.ewelists.com


## Staging and Production Deployment
Deployments to the web and services layers of the staging and production applications are handled by a CI/CD pipeline.

### Setup Staging Infrastructure
1. **Auth Stack:**
    ```
    aws cloudformation create-stack --stack-name Auth-Tools-Staging \
     --template-body file://auth.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --enable-termination-protection \
     --parameters ParameterKey=Environment,ParameterValue=staging
    ```
1. **Hosted Zone:**
    ```
    aws cloudformation create-stack --stack-name Web-HostedZone-Tools-Staging --template-body file://web-hostedzone.yaml \
      --parameters ParameterKey=Environment,ParameterValue=staging
    ```
1. **SSL Certificate:** (Get HostedZoneID from outputs of HostedZone stack, must be in us-east-1 region for use with CloudFront)
    ```
    aws cloudformation create-stack --region us-east-1 --stack-name Web-SSL-Tools-Staging \
    --template-body file://web-sslcert.yaml \
    --parameters ParameterKey=Environment,ParameterValue=staging \
      ParameterKey=HostedZone,ParameterValue=ABCDEFGHIJKL123456789

    aws ssm put-parameter --name /tools.ewelists.com/staging/SSLCertificateId --type String \
     --value "f38ecd9a-...."
    ```
1. **Setup User:**
    ```
    aws cognito-idp sign-up --region eu-west-1 --client-id YOUR_COGNITO_APP_CLIENT_ID --username admin@example.com --password Passw0rd!
    aws cognito-idp admin-confirm-sign-up --region eu-west-1 --user-pool-id YOUR_COGNITO_USER_POOL_ID --username admin@example.com
    ```

### Setup Production Infrastructure
1. **Auth Stack:** Create Auth stack
    ```
    aws cloudformation create-stack --stack-name Auth-Tools-Prod \
     --template-body file://auth.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --enable-termination-protection \
     --parameters ParameterKey=Environment,ParameterValue=prod
    ```
1. **Hosted Zone:**
    ```
    aws cloudformation create-stack --stack-name Web-HostedZone-Tools-Prod --template-body file://web-hostedzone.yaml \
      --parameters ParameterKey=Environment,ParameterValue=prod
    ```
1. **SSL Certificate:** (Get HostedZoneID from outputs of HostedZone stack, must be in us-east-1 region for use with CloudFront)
    ```
    aws cloudformation create-stack --region us-east-1 --stack-name Web-SSL-Tools-Prod \
    --template-body file://web-sslcert.yaml \
    --parameters ParameterKey=Environment,ParameterValue=prod \
      ParameterKey=HostedZone,ParameterValue=ABCDEFGHIJKL123456789

    aws ssm put-parameter --name /tools.ewelists.com/prod/SSLCertificateId --type String \
     --value "f38ecd9a-...."
    ```
1. **Setup User:**
    ```
    aws cognito-idp sign-up --region eu-west-1 --client-id YOUR_COGNITO_APP_CLIENT_ID --username admin@example.com --password Passw0rd!
    aws cognito-idp admin-confirm-sign-up --region eu-west-1 --user-pool-id YOUR_COGNITO_USER_POOL_ID --username admin@example.com
    ```

### Setup Pipeline and deploy
1. **Postman Collection:** (Environment Ids should already exist, but we need to add the Tools Collection ID.  The Environment IDs should have been created by the Main application setup.  See [Postman](https://github.com/Alex-Burgess/ewelists.com/blob/master/documentation/reference.md#postman) for reference commands for retrieving IDs.)
    ```
    aws ssm put-parameter --name /Postman/Collection/Tools --type String --value "6596444-38afc6ee-????"
    ```
1. **Create pipeline:**
    ```
    aws cloudformation create-stack --stack-name Pipeline-Tools  \
      --template-body file://pipeline.yaml  \
      --capabilities CAPABILITY_NAMED_IAM  \
      --parameters ParameterKey=GitHubToken,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github" --with-decryption --query 'Parameter.Value' --output text` \
        ParameterKey=GitHubSecret,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github_secret" --with-decryption --query  'Parameter.Value' --output text`
    ```
1. **Initial Deploy:** The **Release Change** button can be used to trigger the first run through the pipeline.
1. **Update Config.js and Re-Deploy:** This step could be further automated in the **Deploy-Web** step, by passing the API details as variables to the build script that deploys the web application and automating the update of the config.js file.

## Reference
### Admin Create Accounts
```
aws cognito-idp sign-up --region eu-west-1 --client-id YOUR_COGNITO_APP_CLIENT_ID --username admin@example.com --password Passw0rd!

aws cognito-idp admin-confirm-sign-up --region eu-west-1 --user-pool-id YOUR_COGNITO_USER_POOL_ID --username admin@example.com
```

### Reset password
```
aws cognito-idp admin-set-user-password --region eu-west-1 --user-pool-id YOUR_COGNITO_USER_POOL_ID --username admin@example.com --password Passw0rd!
```

### Update Pipeline
```
aws cloudformation update-stack --stack-name Pipeline-Tools  \
  --template-body file://pipeline.yaml  \
  --capabilities CAPABILITY_NAMED_IAM  \
  --parameters ParameterKey=GitHubToken,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github" --with-decryption --query 'Parameter.Value' --output text` \
    ParameterKey=GitHubSecret,ParameterValue=`aws ssm get-parameter --name "/ewelists.com/github_secret" --with-decryption --query  'Parameter.Value' --output text`
```
