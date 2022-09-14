
# Multi-Region CockroachDB Dedicated Logs in CloudWatch: 
##  A "how to guide" on setting up log shipping from a CockroachDB Dedicated Multi-Region Cluster to AWS CloudWatch Logs

# Introduction
An enterprise level organization with strict security protocols may want their logs from a CockroachDB Dedicated cluster, hosted in AWS, to be available in CloudWatch for Security and Information Management (SIEM).  Or there may be a need to proactively monitor logs for errors or events that need to be addressed by the organization.  Whatever the reason, logs from a CockroachDB Dedicated cluster can be made available in AWS CloudWatch.  Logs are also avialable in Cloud Logging for GCP Hosted Dedicated Clusters.  

This "How-To" is intended to help you get your logs from a multi-region CockroachDB Dedicated cluster into AWS CloudWatch.    The [Cockroach Labs Documentation](https://www.cockroachlabs.com/docs/cockroachcloud/export-logs.html) does a great job of explaining how to perform this function for a single-region cluster.   This is for those folks setting log shipping for a multi-region cluster.  

We'll assume you already have a CockroachDB Dedicated cluster, a database user, and a terminal session with access to the cluster.

## High-Level Architecture
![High-Level Architecture Diagram](Resources/crdb-cloudwatch-logs.jpg)
</br></br>
The multi-region CockroachDB Dedicated Cluster will be in a dedicated Cockroach AWS Account.  Logs will be "shipped" to the customer account and will stay "in-region"; that is, logs generated in a region, say us-west-2, will stay in that region (us-west-2).  In a multi-region cluster, you will have logs in the same regions of CloudWatch that you have Cockroach Instances.  

# Steps to Generate Logs in CloudWatch

To enable CloudWatch Logging, we're going to need to gather the following list of items (all of which we will do below).  I highly suggest keeping track of each one of these as it is created; it will save you time from having to go back and look them up.  

- The name of your CloudWatch Log Groups
- The ARN(s) of your CloudWatch Log Groups (which we will create below).  You should have one ARN for each log group created (in our example, we'll have 3 ARNs).
- Your Cockroach Organization ID (From the Cockroach Cloud UI Settings Page)
- The Secret Key of your service account (from the Cockroach Cloud UI Access Page) in order to access the API.
- AWS Account Id of your dedicated cluster (from an API call)
- Cluster Id of your dedicated cluster (from an API call)
- The ARN of your AWS IAM Policy 
- The ARN of your AWS IAM Role 

# Step 1 - Create the AWS CloudWatch log group in each region
**In each region** of your CockroachDB Cluster, create a log group in AWS CloudWatch.  *Be sure to name the log group exactly the same in each region.*  Save the name of the log group, we'll need that in step 7.  To keep things easy to manage, I include the cluster name "nollen-cloudwatch-cluster" in the log group name.  In my case, my multi-region cluster is in us-east-1, us-east-2 and us-west-2 so I'll create 3 log groups, one in each of those regions.  
</br>
![Create Log Group](Resources/Create-Log-Group.JPG) </br></br>
By clicking on the log group name you just created, a summary will be presented which includes the ARN.  Copy the ARN from each region -- we'll need those later when we create the IAM Policy.  

![Create Log Group](Resources/Create-Log-Group-Confirmation.JPG) </br></br>

# Step 2 - Create a Service Account and "Secret Key" in the Cockroach Cloud UI.
## Create The Service Account 
The service account is required to use the API.  The first step  is to navigate to the "Access" page (left hand column) of the Organization page in the CockroachDB Cloud console, select the Service Accounts Tab and select the `Create Service Account` button.
![Access Page of the CockroachDB Cloud console](Resources/Create-Service-Account-Start.JPG) </br></br>

Next enter an "Account name", "Description" and Permissions" and select the "Create Button</br>
![Create Service Account Step 2](Resources/Create-Service-Account-Step-1.JPG) </br></br>
Now enter the "API key name" and select the "Create" button. </br>
![Create Service Account Step 3](Resources/Create-Service-Account-Step-2.JPG)
</br></br>
In the final step, copy the "Secret key" and keep that handy (and safe).  We will be using this to call the Cockroach Cloud API.</br>
![Create Service Account Step 4](Resources/Create-Service-Account-Step-3.JPG)
</br></br>

Now that we have a "Secret Key", we can call the API to get the AWS Account Number and Cluster ID associated with our CockroachDB Dedicated cluster. </br></br>

# Step 3 -  Find the CockroachDB Dedicated AWS Account ID and Cluster ID
To find the Account Id, we'll need to use the API and in order to use the API, we need use the "Secret Key" we created in Step #2. </br>

In a terminal session that has access to the cluster, execute the following command:

```
curl --request GET \
  --url 'https://cockroachlabs.cloud/api/v1/clusters?show_inactive=false&pagination.start=1&pagination.limit=100&pagination.order=ASC' \
  --header "Authorization: Bearer {Secret key}"
```

The API will list all of the Clusters in your organization.  Find the cluster name for which you want to set up CloudWatch logging and find the "id" and "account_id" values.  For example:

```
    {
      "id": "14835272-8c3b-4b7a-ad3a-d4de878f4e34",
      "name": "nollen-cloudwatch-cluster",
      "cockroach_version": "v22.1.6",
      "plan": "DEDICATED",
      "cloud_provider": "AWS",
      "account_id": "123456789012",
      "state": "CREATED",
      ...
    }
```
In this example, the Cluster Id (id) is `14835272-8c3b-4b7a-ad3a-d4de878f4e34` and the AWS Account Id (account_id) is `123456789012`.  You'll need both of these in the next steps, so keep track of them.

# Step 4 - Find Your Cockroach Organization ID
From the Cockroach Cloud UI, from the "Settings" tab on the left hand column, you'll find the Organizaiton ID; it should be 36 numbers including 4 dashes.  
![Get The Cockroach Organization ID](Resources/Get-Organization-ID.JPG)

# Step 5 - Create an IAM Policy
Earlier, when we created the Log groups in CloudWatch, we were presented with CloudWatch Log Group ARNs.  We'll use those ARNs now to create a policy.  

In AWS IAM, choose "Policy" from the left hand tabs and "Create Policy".

Copy the JSON below and paste it into the "Create Policy" section in the "JSON" tab.  Replace the 3 ARNs of the Resource seciont in the example below with the ARN(s) of your CloudWatch Log Groups (we created those in step 1).

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams",
                "logs:PutRetentionPolicy",
                "logs:PutLogEvents"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:logs:us-west-2:123456789012:log-group:/cockroachdb/nollen-cloudwatch-cluster/logs:*",
                "arn:aws:logs:us-east-1:123456789012:log-group:/cockroachdb/nollen-cloudwatch-cluster/logs:*",
                "arn:aws:logs:us-east-2:123456789012:log-group:/cockroachdb/nollen-cloudwatch-cluster/logs:*"
            ]
        }
    ]
}
```

![AWS IAM Create Policy](Resources/AWS-IAM-Create-Policy.jpg)
</br></br>
Finally,  save the policy.  
![Create IAM Policy Final Step](Resources/AWS-IAM-Create-Policy-Final-Step.jpg)

# Step 6 - Create A Cross Account IAM Role
In this step, we're going to create an AWS IAM Role that will be used by Cockroach Labs to move logs from the dedicated account running the CockroachDB Dedicated Cluster to your account.  In this role, we're going to specify the account that can use the role along with the policy we created in the previous step which allows access to our CloudWatch Log Groups.</br></br>
In AWS IAM, create a new role.  The key elements are:
- The "Trusted entity type" is "AWS account"
- The "Account ID" to supply is the AWS Account ID from step 3 (This is the "account_id")
 </br></br>

![Create Account Role Step 1](Resources/AWS-IAM-Create-Role.JPG)
</br></br>
In permissions add the policy we created in step 5 </br></br>
![Add IAM Policy to IAM Role](Resources/AWS-IAM-Create-Role-Add-Policy.JPG)

</br></br>
And finally, name and save the role.  
![Name and Save the Role](Resources/AWS-IAM-Create-Role-Name-Role.JPG)
</br></br>
Be sure to save the ARN (Amazon Resource Name) of the role you just created.   We'll need that in the next step.


# Step 7 - Call the API to Start Log Shipping
In this step we're going to need the following information to build the API call:
- log_group_name:  from step 1 when we created the CloudWatch Log Groups
- secret_key: created in step 2
- cluster_id:  this is from step 3 and is the "id" from the API response
- role_arn: this was created in step 6  

```
curl --request POST \
  --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/logexport \
  --header "Authorization: Bearer {secret_key}" \
  --data '{"type": "AWS_CLOUDWATCH", "log_name": "{log_group_name}", "auth_principal": "{role_arn}"}'
```

The response I received:
```
{
  "cluster_id": "{my cluster ID}",
  "status": "ENABLING",
  "user_message": "",
  "spec": {
    "type": "AWS_CLOUDWATCH",
    "log_name": "/cockroachdb/nollen-cloudwatch-cluster/logs",
    "auth_principal": "arn:aws:iam::{my account ID}:role/nollen-log-export-role"
  },
  "created_at": "2022-09-13T19:08:21.063388Z",
  "updated_at": "2022-09-13T19:08:21.063388Z"
}
```

To check on the status
```
curl --request GET \
  --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/logexport \
  --header "Authorization: Bearer {secret_key}"
```

When the process is complete the response will look like this:
```
{
  "cluster_id": "{my cluster ID}",
  "status": "ENABLED",
  "user_message": "",
  "spec": {
    "type": "AWS_CLOUDWATCH",
    "log_name": "/cockroachdb/nollen-cloudwatch-cluster/logs",
    "auth_principal": "arn:aws:iam::{my account ID}:role/nollen-log-export-role"
  },
  "created_at": "2022-09-13T19:08:21.063388Z",
  "updated_at": "2022-09-13T19:42:38.924969Z"
}
```

This takes a while -- all the nodes of the cluster need to be restarted.  

And in CloudWatch Logs you'll be able to operate on the logs in each region:
![CloudWatch Log Shipping Complete](Resources/cloud-watch-log-shipping-complete.JPG)


# Disable Log Shipping
If you need to disable log shipping once it has been enabled, use the API:

```
  curl --request DELETE \
  --url https://cockroachlabs.cloud/api/v1/clusters/{Cluster ID}/logexport \
  --header "Authorization: Bearer {Secret Key}"
```


-------------------------------------------------------------








curl --request GET \
  --url https://cockroachlabs.cloud/api/v1/clusters/14835272-8c3b-4b7a-ad3a-d4de878f4e34/logexport \
  --header "Authorization: Bearer CCDB1_AIYybfcRQhkFsmaHkOQoj2_Png19xaVlLqedBMZdWmIj1RMzwbbQ8sgWLmSgI4W"


curl --request POST \
  --url https://cockroachlabs.cloud/api/v1/clusters/14835272-8c3b-4b7a-ad3a-d4de878f4e34/logexport \
  --header "Authorization: Bearer CCDB1_AIYybfcRQhkFsmaHkOQoj2_Png19xaVlLqedBMZdWmIj1RMzwbbQ8sgWLmSgI4W" \
  --data '{"type": "AWS_CLOUDWATCH", "log_name": "/cockroachdb/nollen-cloudwatch-cluster/logs", "auth_principal": "arn:aws:iam::{my account id}:role/nollen-log-export-role"}'













```
curl --request GET \
  --url https://cockroachlabs.cloud/api/v1/clusters/%7B{Your Cluster ID Here}%7D \
  --header 'Authorization: Bearer {Your "Secret Key" Here}'

curl --request GET \
  --url https://cockroachlabs.cloud/api/v1/clusters/%7B1d4d68ed-a173-461e-a522-4fbca2b062e1%7D \
  --header 'Authorization: Bearer CCDB1_AIYybfcRQhkFsmaHkOQoj2_Png19xaVlLqedBMZdWmIj1RMzwbbQ8sgWLmSgI4W'


```
Cloudwatch Log Group
/cockroachdb/nollen-cloudwatch-cluster/logs
arn:aws:logs:us-west-2:{my account id}:log-group:/cockroachdb/nollen-cloudwatch-cluster/logs:*

Role ARN
arn:aws:iam::{my account id}:role/nollen-log-export-role

Organization Id
1d4d68ed-a173-461e-a522-4fbca2b062e1

Cluster Id
1d4d68ed-a173-461e-a522-4fbca2b062e1


Get Secret
CCDB1_AIYybfcRQhkFsmaHkOQoj2_Png19xaVlLqedBMZdWmIj1RMzwbbQ8sgWLmSgI4W

Password for Ron
ron123

cockroach sql --url "postgresql://ron:ifod1fGT9MICJWnblt5qQw@internal-nollen-cloudwatch-cluster-6dn.aws-us-west-2.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full&sslrootcert=$HOME/Library/CockroachCloud/certs/nollen-cloudwatch-cluster-ca.crt"

curl --request GET \
  --url 'https://cockroachlabs.cloud/api/v1/clusters?show_inactive=false&pagination.start=1&pagination.limit=100&pagination.order=ASC' \
  --header "Authorization: Bearer CCDB1_AIYybfcRQhkFsmaHkOQoj2_Png19xaVlLqedBMZdWmIj1RMzwbbQ8sgWLmSgI4W"



    {
      "id": "14835272-8c3b-4b7a-ad3a-d4de878f4e34",
      "name": "nollen-cloudwatch-cluster",
      "cockroach_version": "v22.1.6",
      "plan": "DEDICATED",
      "cloud_provider": "AWS",
      "account_id": "123456789012",
      "state": "CREATED",
      "creator_id": "bd445dd7-8ac1-4d75-82b9-6e062d3f1cf4",
