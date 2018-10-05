## AWS Config Rule for checking users in Active Directory

AWS Config Rule is a service that provides managed and customized compliance checks in AWS environment. It's designed to work natively with various of AWS resources. Additionally, it also has capability to extend coverage to unsupported resources using customized rules.

This repo provides a solution to integrate AWS Config Rules with application resources. Specifically, the example stack will have a compliance check for passwords of users in Active Directory.

### What the rule checks
The rule checks all users' password configuration in Active Directory. For normal users, it reports non_compliant for the ones who have "password never expires" enabled. For service accounts, it allows "password never expires" but will report non_compliant for password age longer than 90 days. (This is common compliance requirement for AD in large companies - human users should not have non-expiring password; for service accounts, password should also be rotated regularly).

### How does it work
Services in use:
- AWS Config - Compliance tool and reporting dashboard
- AWS SSM - As the check requires action on application level, SSM is used to perform the logic
- AWS Lambda - A customized Config rule will invokes a Lambda function to perform checks. In this case, Lambda will run an SSM document and leverage SSM to do the task
- Powershell with AWS SDK - In this case of checking AD resources, Powershell is used with AWS SDK for reporting back to Config. If you want to check stuff on a Linux server and report back to Config, you can make it in Shell.

Simple work flow:
1. Config
