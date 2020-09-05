## Project Overview

This AWS architecture automates the process of generating DynamoDB backup logs and stores the logs in the CloudWatch Logs service. This solution is useful in situations where a system is needed to track incoming data for oversight or compliance reasons. A slight modification like integrating an S3 bucket can create a cost-efficient daily DynamoDB backup store for data processing and storage.

![diagram](/img/diagram.png)

### Resources Used:

- AWS VPC (configured to connect to SSH)
- AWS CloudWatch
- AWS Cloud9 IDE
- AWS Lambda
- AWS DynamoDB
- AWS S3
- Python 3.7
- Boto3 v3.6


### File Descriptions:

- [MoviesCreateTable.py](https://github.com/lizgarseeyah/AWS-Scheduling-DB-Backups/blob/master/DynamoDB/MoviesCreateTable.py) - python code to create a ‘Movies’ table in DynamoDB
- [MoviesLoadData.py](https://github.com/lizgarseeyah/AWS-Scheduling-DB-Backups/blob/master/DynamoDB/MoviesLoadData.py) - python code to load data into DynamoDB
- [lambda-role-permissions.json](https://github.com/lizgarseeyah/AWS-Scheduling-DB-Backups/blob/master/lambda-role-permissions.json) - JSON code to add to the function role metadata
- [lambda_function.py](https://github.com/lizgarseeyah/AWS-Scheduling-DB-Backups/blob/master/lambda_function.py) - python code to handle backups
- [load-data-instructions.md](https://github.com/lizgarseeyah/AWS-Scheduling-DB-Backups/blob/master/load-data-instructions.md) - Walk-through on how to add data into the database in a shell envrionment.

### Pre-Lab Configuration:

- S3 bucket - backup-logs-dydb
- DynamoDB table - Movies
- VPC (based on Cloud9 IDE configuration)

## Procedure:

- Create a Cloud9 environment to load the data into the database. See “load-data-instructions.md” file for a walk-through.

**Setup a Lambda function**

  - In the AWS Management Console, navigate to the Lambda Service.
  - Click "Create Function”
  - Select “Author from Scratch”, then configure the following settings:
    - Name: “Dynamo-DB-Backup”
    - Runtime: Python 3.7
    - Choose or create an execution role:
      - Select “Create a new role from AWS policy templates”
        - Role name: Lambada-role
        - Policy Templates: None
        
**Edit the Lambda Role to allow permissions to handle events and backups**

  - Navigate to the IAM service, and select the “Lambda-role” that was created in the previous step. This will take you to the “Summary” page.
  - Under the “Permissions” tab, select the “Policy Name”.
  - Select “Edit policy” and under the JSON tab, modify the JSON code to

```markdown

{
"Version": "2012-10-17",

"Statement": [

{
	"Effect": "Allow",

	"Action": [
		"logs:CreateLogGroup",
		"logs:CreateLogStream",
		"logs:PutLogEvents"
],
"Resource": "arn:aws:logs:*:*:*"
},

{

	"Action": [
		"dynamodb:CreateBackup",
		"dynamodb:DeleteBackup",
		"dynamodb:ListBackups"
	],
	"Effect": "Allow",
	"Resource": "*"
	}
]
}
```

  - Select “Review Policy” and then “Save Changes”.
  
- Add the python script to the Lambda Function in the Function code editor from the “lambda_function.py” file. This script has two main functions: create_backup and delete_old_backups. The lambda function will trigger this script to start the backup process.
 
```markdown
import boto3
from datetime import datetime

MAX_BACKUPS = 3  # maximum number of backups to retain

dynamo = boto3.client('dynamodb')


def lambda_handler(event, context):
    if 'TableName' not in event:
        raise Exception("No table name specified.")
    table_name = event['TableName']

    create_backup(table_name)
    delete_old_backups(table_name)


def create_backup(table_name):
    print("Backing up table:", table_name)
    backup_name = table_name + '-' + datetime.now().strftime('%Y%m%d%H%M%S')

    response = dynamo.create_backup(
        TableName=table_name, BackupName=backup_name)

    print(f"Created backup {response['BackupDetails']['BackupName']}")


def delete_old_backups(table_name):
    print("Deleting old backups for table:", table_name)

    backups = dynamo.list_backups(TableName=table_name)

    backup_count = len(backups['BackupSummaries'])
    print('Total backup count:', backup_count)

    if backup_count <= MAX_BACKUPS:
        print("No stale backups. Exiting.")
        return

    # Backups in date descending order (newest to oldest)
    sorted_list = sorted(backups['BackupSummaries'],
                         key=lambda k: k['BackupCreationDateTime'], reverse=True)

    old_backups = sorted_list[MAX_BACKUPS:]

    print(f'Old backups: {old_backups}')

    for backup in old_backups:
        arn = backup['BackupArn']
        print("ARN to delete: " + arn)
        deleted_arn = dynamo.delete_backup(BackupArn=arn)
        backup_name = deleted_arn['BackupDescription']['BackupDetails']['BackupName']
        status = deleted_arn['BackupDescription']['BackupDetails']['BackupStatus']
        print(f'BackupName: {backup_name}, Status: {status}')

    return


if __name__ == "__main__":
    event = {"TableName": "Movies"}
    lambda_handler(event, {})
```

  - The “MAX_BACKUPS = 3” is the number of maximum backups to retain.
Click “Save” once completed.

**Create a Test Event**
- While in the same window, the “Dynamo-DB-Backup” Lambda summary page,  select the “Test” button at the  top.
  - Event Name: Test
  - Contents: { “TableName”:”Movies”}
- Select “Create”
- Select “Test”

![accuracy-f-score](/img/one.png) 

A backup has now been created:

![accuracy-f-score](/img/two.png) 

**Creating a CloudWatch Rule to trigger the Lambda Function.**

- In CloudWatch, select “Create rule”
- Under Event Source, click “Schedule”:
  - Fixed rate of: 1 Minutes
- Under “Targets”, click “Add target”.
- Under”Lambda function”:
  - Function: create-dynamo-DBBackup
  - Configure input: Constant (JSON text)
- In the text box under “Constant (JSON text)”, type {"TableName": "Movies"}.
- Click “Configure details”
- Name: "BackupDynamoDBPerson".
- Click ”Create rule”

![accuracy-f-score](/img/three.png) 

- In the left sidebar, click “Logs”
- Click the create-dynamo-DBBackup log group to open it.
- Click the name of the log stream to open it.
- We have in CloudWatch logs, 3 backup logs based on the code snippet: “MAX_BACKUPS = 3”.

![accuracy-f-score](/img/four.png) 

