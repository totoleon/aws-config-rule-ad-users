## AWS Config Rule for checking users in Active Directory

AWS Config Rule is a service that provides managed and customized compliance checks in AWS environment. It's designed to work natively with various of AWS resources. Additionally, it also has capability to extend coverage to unsupported resources using customized rules.

This repo provides a solution to integrate AWS Config Rules with application resources. Specifically, the example stack will have a compliance check for passwords of users in Active Directory.

### What compliance rule does it check
The rule checks all users' password configuration in Active Directory. For normal users, it reports non_compliant for the ones who have "password never expires" enabled. For service accounts, it allows "password never expires" but will report non_compliant for password age longer than 90 days. (This is common compliance requirement for AD in large companies - human users should not have non-expiring password; for service accounts, password should also be rotated regularly).

### How does it work
![alt text](https://raw.githubusercontent.com/totoleon/aws-config-rule-ad-users/master/doc/Diagram.png)
Services in use:
- AWS Config - Compliance tool and reporting dashboard
- AWS SSM - As the check requires action on application level, SSM is used to perform the logic
- AWS Lambda - A customized Config rule will invokes a Lambda function to perform checks. In this case, Lambda will run an SSM document and leverage SSM to do the task
- Powershell with AWS SDK - In this case of checking AD resources, Powershell is used with AWS SDK for reporting back to Config. If you want to check stuff on a Linux server and report back to Config, you can make it in Shell.

Simple work flow:
1. ConfiG Rule periodic evaluation triggers a Lambda function. The event will contain a `resultToken` for Lambda to report compliance results back to Config.
2. The Lambda function runs an SSM document with giving `resultToken` and `ruleName` as parameters.
3. SSM agent running on the EC2 instance will execute PowerShell commands, which clean up the previous evaluation results and perform new checks.
4. As part of the PowerShell function, it reports compliance results back to Config using the `resultToken`.

### Prerequisites
[ ] Make sure you already have Windows instances as Domain Controllers. The stack does not launch EC2 instances with Active Directory.  
[ ] The Lambda runs SSM targeting instances based on Tags. It runs the document on the instances which are tagged as `Application: ActiveDirectory`  
[ ] The PowerShell script only runs on Primary Domain Controller. It will skip if the DC is not PDC. Make sure your PDC has the identifier tagging. Or you can modify the script to run based on instance Ids.  
[ ] Make sure your instance has PowerShell AWS SDK installed - https://docs.aws.amazon.com/powershell/latest/userguide/pstools-getting-set-up-windows.html  
[ ] Make sure your EC2 instance has SSM agent installed and running - https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-install-ssm-win.html  
[ ] Make sure the EC2 instance has sufficient permission to put evaluation results to the Config Rule. You can associate the policy created in the stack to the Instance Profile of your DC.  
[ ] Make sure you have AWS Config is enabled  

### Deployment steps
1. Launch a stack from `cfn_template.yml` in the region where your Active Directory instances run.
2. Tag your domain controllers with `Application: ActiveDirectory`. The tagged instances will be the target of SSM send command. However, the actual evaluation will only be performed by the Primary Domain Controller.
3. If your domain controller instance doesn't have an Instance Profile, create one and associate. Then, make sure the Instance Role has sufficient permission to perform `config:PutEvaluations` and `config:DeleteEvaluationResults`. You can attach the managed policy created by the stack to your Instance Profile.
4. The Config rule should be deployed and alive. You can trigger it manually in Config console. The result will be like this: ![alt text](https://raw.githubusercontent.com/totoleon/aws-config-rule-ad-users/master/doc/evaluation_results.png)

### CloudFormation stack components
The CloudFormation template includes:
- A Config Rule
- A Lambda function in Python 3.6
- An SSM document Powershell
- IAM Role and permissions for Lambda to invoke the SSM document
- An IAM policy for AD's instance role to report evaluation result

#### Lambda logic
```
import json, boto3, logging
from pprint import pformat
logger = logging.getLogger()
logger.setLevel(logging.INFO)
ssm = boto3.client('ssm')

# The Lambda will be invoked by Config
def lambda_handler(event, context):
    logger.info("Starting evaluation. Received event from Config:")
    logger.info(pformat(event))

    # Get resultToken from the event.
    evaluation_token = event['resultToken']
    rule_region = event['configRuleArn'].split(':')[3]
    rule_name = event['configRuleName']
    rule_params = json.loads(event['ruleParameters'])
    # Get the document name from rule parameters
    document_name = rule_params['EVALUATION_DOC']

    # Invoke the SSM document and pass the resultToken to PowerShell
    command_execution = ssm.send_command(
        Targets=[{"Key":"tag:Application","Values":["ActiveDirectory"]}],
        DocumentName=document_name,
        Comment='Check AD user compliance',
        Parameters={"executionTimeout":["600"],"resultToken":[evaluation_token],"ruleName":[rule_name],"ruleRegion":[rule_region]})
    logger.info('SSM document run command has been sent to DCs. CommandId:' + command_execution['Command']['CommandId'])
```
#### PowerShell Logic:

```
Set-DefaultAWSRegion -Region "{{ ruleRegion }}"
$dc = Get-ADDomainController
$pdc = Get-ADDomainController -Discover -Domain $dc.Domain -Service "PrimaryDC"
if ($dc.Name -eq $pdc.Name) {
  $token = "{{ resultToken }}"
  Import-Module AWSPowerShell

  # Get all normal users and service accounts distinguished by if starting with 'service_'
  $normalUsers = Get-Aduser -Filter 'Name -notLike "service_*"' -Properties Name, PasswordNeverExpires, PasswordLastSet
  # Human users should not have PasswordNeverExpires = True
  $badNonServiceUsers = $normalUsers | where({ $_.PasswordNeverExpires })
  $serviceUsers = Get-Aduser -Filter 'Name -Like "service_*"' -Properties Name, PasswordNeverExpires, PasswordLastSet
  $allowedPasswdAge = 90
  $dateToday = Get-Date
  # Service accounts should have password age less than 90 days
  $badServiceUsers = $serviceUsers | where({ (New-TimeSpan -Start $_.PasswordLastSet -end $dateToday).days -gt $allowedPasswdAge })
  $allUsers = $normalUsers + $serviceUsers
  $badUsers = $badNonServiceUsers + $badServiceUsers
  Remove-CFGEvaluationResult -ConfigRuleName "{{ ruleName }}"
  # Loop through all users and report the evaluation results to AWS Config with given token
  Foreach ($user in $allUsers)
  {
    $complianceType = If ($badUsers -contains $user) {"NON_COMPLIANT"} Else {"COMPLIANT"}
    $evaluation = New-Object Amazon.ConfigService.Model.Evaluation
    $evaluation.ComplianceResourceId = $user.Name
    $evaluation.ComplianceResourceType = 'AWS::::Account'
    $evaluation.ComplianceType = $complianceType
    $evaluation.OrderingTimestamp = Get-Date
    Write-CFGEvaluation -Evaluation $evaluation -ResultToken $token
  }
}
```
