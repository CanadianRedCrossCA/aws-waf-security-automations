# AWS WAF Security Automations customized by Tesera for Red Cross 
This repo is a clone of the original AWS WAF Security Automations solution, and adds few rules for strengthening the WAF for Red Cross

## Existing Rules - https://docs.aws.amazon.com/solutions/latest/aws-waf-security-automations/architecture.html
 - Whitelist Rule
 - Blacklist Rule
 - XSS Rule
 - HTTP Flood Rule
 - Scanners & Probes Rule
 - WAF IP Reputation Lists Rule
 - Bad Bot Rule

## Additional Rules
 - Blacklist bad User-Agent rule - a list of User-Agent headers that is going to be blocked;
 - Size restriction rule - Mitigate abnormal requests via size restrictions. Enforce consistent request hygene, limit size of key elements;
 - Geo match rule - Allow access only within Canada.

## Build
Prerequisites - To build the lambdas used in this solution you'll need:
 - Node.js 10.x
 - Python 3.8

When modifications are required run the following command from the `deployment` folder to create a new build:
 - `./build-s3-dist.sh emis-aws-waf-security-automations-template emis-aws-waf-code EMIsAWSWAFSecurityAutomations 2.3.2`
 If the version of this solution needs to be updated the number `2.3.2` can be changed to the desired new version.
 This command will create (or replace the content of) two subfolders in the `deployment` folder:
  - `global-s3-assets` - contains the cloudformation templates;
  - `regional-s3-assets` - contains the source code for the lambda functions used by this solution.


## Upload the build to S3 
To deploy the build you need to execute the following commands from the `deployment` folder:
  `aws s3 cp --recursive global-s3-assets/ s3://emis-aws-waf-security-automations-template/EMIsAWSWAFSecurityAutomations/2.3.2/ --profile redcross`
 
and 

  `aws s3 cp --recursive regional-s3-assets/ s3://emis-aws-waf-code-ca-central-1/EMIsAWSWAFSecurityAutomations/2.3.2/ --profile redcross`

If the version has changed the new version should be used instead of `2.3.2` in both `aws s3 cp` commands.

## Limitations
 - Cloudformation doesn't support `AWS::WAF::GeoMatchSet`, but it only supports `AWS::WAFRegional::GeoMatchSet`. For the GeoMatch Rule we needed to create manually a GeoMatchSet to be used with the rule. Our implementation relies on GeoMatchSet with id `f8d2a474-fad5-4b71-b95e-76778cf4859f` to exist.

 - WAF Classic allows only up to 10 rules per ACL - currently we are maxing out the number of rules. We opened a quota increase ticket and after quite lengthy communication with AWS Support. Even though the support case was approved, the limitations is still in place, so at the moment we can not add new rules without removing an existing one.

 - WAF Classic has a limitation of up to 5 "Rate Rules" per account and per region. Since all of our environments are in the same account and region, and we have 10 Web ACLs in total, we can not have a separate HTTP Flood Rule in all of the Web ACLs. This forced us to share the API Flood Rule with the APP Web ACL. See the Deployment section below for details how to apply the sharing.

## Deployment
After the build is uploaded to S3, we can use it to deploy the solutions. 
The deployment can be done from the AWS Management Console -> CloudFormation.
Make sure you are located in the ca-central-1 region.

Select Create Stack -> with new Resources.

Select "Template is ready"
For template source select "Amazon S3 URL", and specify the location of the template like - `https://emis-aws-waf-security-automations-template.s3.ca-central-1.amazonaws.com/EMIsAWSWAFSecurityAutomations/2.3.2/aws-waf-security-automations.template`

Click "Next"

In each Environment Dev, QA, UAT, Train and Prod we have two Web ACLs - one for the APP and one for the API, both attached to the CloudFront distributions.
The current names of the Web ACLs are:

dev-app-ca-central-1-AWSWAFSecurityAutomations
dev-api-ca-central-1-AWSWAFSecurityAutomations

qa-app-ca-central-1-AWSWAFSecurityAutomations
qa-api-ca-central-1-AWSWAFSecurityAutomations

uat-app-ca-central-1-AWSWAFSecurityAutomations
uat-api-ca-central-1-AWSWAFSecurityAutomations

train-app-ca-central-1-AWSWAFSecurityAutomations
train-api-ca-central-1-AWSWAFSecurityAutomations

prod-app-ca-central-1-AWSWAFSecurityAutomations
prod-api-ca-central-1-AWSWAFSecurityAutomations

Use one of these names, or if you are creating a new Web ACL while Web ACL with the same name exists, you can add a suffix after the name like "dev-app-ca-central-1-AWSWAFSecurityAutomations1".

For "Application Access Log Bucket Name" you need to specify the bucket which contains the CloudFront logs. The current buckets are:

dev-emis-registration-app-access-logs
dev-emis-registration-api-access-logs

qa-emis-registration-app-access-logs
qa-emis-registration-api-access-logs

uat-emis-registration-app-access-logs
uat-emis-registration-api-access-logs

train-emis-registration-app-access-logs
train-emis-registration-api-access-logs

prod-emis-registration-app-access-logs
prod-emis-registration-api-access-logs

Leave all other options as they are.

Click "Next"

In the Tags section, add tags to specify the application and the environment like:
Application  = app
Environment = dev

Click "Next"

Check both checkboxes:
I acknowledge that AWS CloudFormation might create IAM resources with custom names.
I acknowledge that AWS CloudFormation might require the following capability: CAPABILITY_AUTO_EXPAND

Click "Create Stack"

Sequence for deploying the Web ACL in APP and API:
1. Create the Web ACL for the APP
2. Remove the "HTTP Flood Rule" from APP Web ACL
3. Create the Web ACL for the API
4. In the APP Web ACL add the "HTTP Flood Rule" from the API Web ACL

After the stacks are created they need to be attached to the CloudFront distributions. This needs to be done in Terraform in the registration-infrastructure repo, by specifying the Web ACL id in all environments for both the APP and the API, like:
https://github.com/CanadianRedCrossCA/registration-infrastructure/blob/master/dev/app/terraform.tfvars#L5
https://github.com/CanadianRedCrossCA/registration-infrastructure/blob/master/dev/api/terraform.tfvars#L15.
