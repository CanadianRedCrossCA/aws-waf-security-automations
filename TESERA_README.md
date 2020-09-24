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
 - Disable disaster/event id rule - this rule will block requests that contain given disaster ids in their URI.

## Build
Prerequisites - To build the lambdas used in this solution you'll need:
 - Python 3.8

When modifications are required run the following command from the `deployment` folder to create a new build:
 - `TEMPLATE_OUTPUT_BUCKET=emis-aws-waf-security-automations`
 - `DIST_OUTPUT_BUCKET=emis-aws-waf-code`
 - `SOLUTION_NAME=aws-waf-security-automations`
 - `VERSION=3.0.0`
 - `AWS_REGION=us-east-1`
 - `./build-s3-dist.sh $TEMPLATE_OUTPUT_BUCKET $DIST_OUTPUT_BUCKET $SOLUTION_NAME $VERSION`
 If the version of this solution needs to be updated the number `3.0.0` can be changed to the desired new version.
 This command will create (or replace the content of) two subfolders in the `deployment` folder:
  - `global-s3-assets` - contains the cloudformation templates;
  - `regional-s3-assets` - contains the source code for the lambda functions used by this solution.


## Upload the build to S3 
To deploy the build you need to execute the following commands from the `deployment` folder:
  `aws s3 cp global-s3-assets s3://$TEMPLATE_OUTPUT_BUCKET/aws-waf-security-automations/$VERSION --recursive --acl bucket-owner-full-control --profile redcross`
 
and 

  `aws s3 cp regional-s3-assets s3://$DIST_OUTPUT_BUCKET-$AWS_REGION/aws-waf-security-automations/$VERSION --recursive --acl bucket-owner-full-control --profile redcross`

## Limitations and issues of the Cloudformation template
 - The Cloudformation resource AWS::WAFv2::IPSet has a requirement for the WAFv2 resources to be created in the US East (N. Virginia) Region, us-east-1 (https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-wafv2-ipset.html). This will enforce us to create the WebACL in us-east-1, and also move the logging bucket for the Cloudfront distributions to us-east-1.

 - There is a limitation for the length of the WebACL name we can pick. Due to the way the Cloudformation template is implemented, when the permissions are created if the name of the template is long, it will be shortened and the roles with permissions that are assigned to the lambda functions endup granting permissions for the incorrect resources. Using names with lengts similar to the examples specified here works fine.

 - The firehose-athena template originally created workgroups with hardcoded names, which is causing duplicate error (name already exsists), so we had to prefix the the workgroup names with the Stack name.

## Deployment
After the build is uploaded to S3, we can use it to deploy the solutions. 
The deployment can be done from the AWS Management Console -> CloudFormation.
Make sure you are located in the us-east-1 region.

Select Create Stack -> with new Resources.

Select "Template is ready"
For template source select "Amazon S3 URL", and specify the location of the template like - `https://emis-aws-waf-security-automations.s3.us-east-1.amazonaws.com/aws-waf-security-automations/3.0.0/aws-waf-security-automations.template`

Click "Next"

In each Environment Dev, QA, UAT, Train and Prod we have two Web ACLs - one for the APP and one for the API, both attached to the CloudFront distributions.
The current names of the Web ACLs are:

dev-app-emis-waf
dev-api-emis-waf

qa-app-emis-waf
qa-api-emis-waf

uat-app-emis-waf
uat-api-emis-waf

train-app-emis-waf
train-api-emis-waf

prod-app-emis-waf
prod-api-emis-waf


For Prod, Train and UAT we are going to use the following rules: 
APP:
 - WhitelistRule
 - BlacklistRule
 - HttpFloodRateBasedRule
 - ScannersAndProbesRule
 - IPReputationListsRule
 - BadBotRule
 - XssRule
 - SizeRule
 - UserAgentRule
 - CanadaOnlyRule
 
API:
 - WhitelistRule
 - BlacklistRule
 - HttpFloodRateBasedRule
 - ScannersAndProbesRule
 - IPReputationListsRule
 - XssRule
 - SizeRule
 - UserAgentRule
 - CanadaOnlyRule
 - BlockDisasterRule

The difference between the APP and the API WAF ACLs is that the BadBotRule doesn't apply in the API WAF and the BlockDisasterRule doesn't apply in the APP WAF.
For "Application Access Log Bucket Name" you need to specify the bucket which contains the CloudFront logs. The current buckets are:

uat-emis-registration-app-access-logs-us-east-1
uat-emis-registration-api-access-logs-us-east-1

train-emis-registration-app-access-logs-us-east-1
train-emis-registration-api-access-logs-us-east-1

prod-emis-registration-app-access-logs-us-east-1
prod-emis-registration-api-access-logs-us-east-1

For PROD and TRAIN the Default Action will be "Allow".

For UAT the Default Action will be changed to "Block" after the WAF ACL is created. The reason to keep all the WAF rules in UAT, but change the Default Action to "Block" is to have an environment where we can run penetration tests. Before a PEN test is run, we are going to be changing the Default Action to "Allow" for the time duration of the PEN test, allowing us to test against the same configurations as in TRAIN and PROD.

For DEV and QA we are going to use only the WhitelistRule and the BlacklistRule rules, since those environments are locked down for only CRC and Tesera access. After the WAF ACLs are created in DEV and QA the Default Action will be changed to "Block".

Click "Next"

In the Tags section, add tags to specify the application and the environment like:
Application  = app
Environment = dev

Click "Next"

Check both checkboxes:
I acknowledge that AWS CloudFormation might create IAM resources with custom names.
I acknowledge that AWS CloudFormation might require the following capability: CAPABILITY_AUTO_EXPAND

Click "Create Stack"

After the stacks are created they need to be attached to the CloudFront distributions. This needs to be done in Terraform in the registration-infrastructure repo, by specifying the Web ACL id in all environments for both the APP and the API, like:
https://github.com/CanadianRedCrossCA/registration-infrastructure/blob/master/dev/app/terraform.tfvars#L5
https://github.com/CanadianRedCrossCA/registration-infrastructure/blob/master/dev/api/terraform.tfvars#L15.

When an update is required to the cloudformation templates or the the code of the lambdas, when a new build is created and uploaded the Cloudformation stacks can be updated instead of recreating fresh all resources. This way there won't be a need to change the Web ACL Id in CloudFront. The update is done throught the Update button on the CloudFormation's Stacks page. The rest of the deployment steps are the same as with creating a new stack.