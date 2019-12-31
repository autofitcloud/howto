# howto/01-aws-spot-and-email

*How to automatically receive an email upon fulfillment of an AWS spot instance request*

Useful for spot requests that take some time to get fulfilled

Triggered from [this reddit discussion](https://www.reddit.com/r/aws/comments/ehm1ah/best_aws_solution_for_high_memory_usage/fckm1g5)


## Usage

The following steps will

- create a `t3.medium` spot instance request in `us-east-1`
- send an email to the configured email address upon fulfillment
- terminate the instance after 60 minutes


Download this repository

```
git clone https://github.com/autofitcloud/howto/ autofitcloud-howto
cd autofitcloud-howto/01-aws-spot-and-email

OR IF YOU DONT HAVE GIT

wget https://github.com/autofitcloud/howto/archive/master.zip -O autofitcloud_howto.zip
unzip autofitcloud_howto.zip
mv howto-master autofitcloud-howto
cd autofitcloud-howto/01-aws-spot-and-email
```

Copy templated files

```
cp user_data.sh.template user_data.sh
cp launch_specification.json.template launch_specification.json
```

Replace variables in curly braces in new files with your own values, eg

```
launch_specification.json:
  KeyName: shadi # use `aws --region=us-east-1 ec2 describe-key-pairs` to list available key names in your account

user_data.sh:
  aws_access_key_id: abcdef # copy from ~/.aws/credentials
  aws_secret_access_key: abcdef # ditto
  to_email: shadi@autofitcloud.com # email to notify on spot instance creation
```


Verify the email that will receive a notification on spot instance creation

```
aws --region us-east-1 ses verify-email-identity --email-address "shadi@autofitcloud.com"
```

Click link in email and get "Congratulations, your email is verified"

Test sending email is working

```
aws --region us-east-1 ses send-email --from "shadi@autofitcloud.com" --to "shadi@autofitcloud.com" --subject "aloha asdf" --text "hey hey hey"
```

Encode the file `user_data.sh` in base64 for its use in `launch_specification.json`

```
base64 user_data.sh -w0 > user_data.sh.b64
```

The base-64 encoded file can also be checked with

```
cat user_data.sh.b64 | base64 --decode
```

Since the `launch_specification.json` doesn't support passing the user-data as a path to a file, eg `file://user_data.sh.b64`,
manually copy the contents of `user_data.sh.b64` to the `UserData` field value in `launch_specification.json`.

Request spot instance and have it send email on fulfillment (remember to use a new `client-token` for different requests)

```
# docs at https://docs.aws.amazon.com/cli/latest/reference/ec2/request-spot-instances.html
aws --region=us-east-1 \
  ec2 request-spot-instances \
  --block-duration-minutes=60 \
  --client-token=test-isitfit-spot-6 \
  --instance-count=1 \
  --spot-price=0.3 \
  --launch-specification file://launch_specification.json
```

Output will be similar to

```
{
    "SpotInstanceRequests": [
        {
            "BlockDurationMinutes": 60,
            "CreateTime": "2019-12-31T07:51:31.000Z",
            "LaunchSpecification": {
                "SecurityGroups": [
                    {
                        "GroupName": "default",
                        "GroupId": "sg-123456"
                    }   
                ],  
                "ImageId": "ami-00a208c7cdba991ea",
                "InstanceType": "t3.medium",
                "KeyName": "shadi",
                "Placement": {
                    "AvailabilityZone": "us-east-1a"
                },  
                "SubnetId": "subnet-abcdef",
                "Monitoring": {
                    "Enabled": false
                }   
            },  
            "ProductDescription": "Linux/UNIX",
            "SpotInstanceRequestId": "sir-123456",  # <<<<<<<<<< Request ID, useful to get Instance ID and public IP address
            "SpotPrice": "0.300000",
            "State": "open",
            "Status": {
                "Code": "pending-evaluation",
                "Message": "Your Spot request has been submitted for review, and is pending evaluation.", # <<<<<<<<<<<< LGTM
                "UpdateTime": "2019-12-31T07:51:31.000Z"
            },
            "Type": "one-time",
            "InstanceInterruptionBehavior": "terminate"
        }
    ]
}
```


Check status of spot request

```
req_id=sir-12345 # copy this from output of above `ec2 request-spot-instances` command, field value of `SpotInstanceRequestId`
aws --region=us-east-1 ec2 describe-spot-instance-requests --spot-instance-request-ids=$req_id
```

Output similar to (check notes in-line in json for further info)

```
{   
    "SpotInstanceRequests": [
        {   
            "ActualBlockHourlyPrice": "0.023000",  # <<<<<<<< Got 2.3 cents per hour, even if specified max price of 30 cents per hour
            "BlockDurationMinutes": 60,
            "CreateTime": "2019-12-31T07:51:31.000Z",
            "InstanceId": "i-123456789",      # <<<<<<<< Instance ID, useful to get public IP address and ssh
            "LaunchSpecification": {
                "SecurityGroups": [
                    {   
                        "GroupName": "default",
                        "GroupId": "sg-123456"
                    }
                ],
                "ImageId": "ami-00a208c7cdba991ea",
                "InstanceType": "t3.medium",
                "KeyName": "shadi",
                "Placement": {
                    "AvailabilityZone": "us-east-1a"
                },
                "SubnetId": "subnet-abcdef",
                "Monitoring": {
                    "Enabled": false
                }
            },
            "LaunchedAvailabilityZone": "us-east-1a",
            "ProductDescription": "Linux/UNIX",
            "SpotInstanceRequestId": "sir-123456",
            "SpotPrice": "0.300000",
            "State": "active",
            "Status": {
                "Code": "fulfilled",
                "Message": "Your spot request is fulfilled.", # <<<<<<<<<<<<< HOORAY!
                "UpdateTime": "2019-12-31T07:51:32.000Z" # <<<<<<<<<<<<<<< Make sure that this is matching with the current date, i.e. command `date -u`.
                                                         # If the client-token field above is forgotten to be updated between requests, this will just return the previously launched spot instance data
            },
            "Tags": [],
            "Type": "one-time",
            "ValidUntil": "2020-01-07T07:51:31.000Z",
            "InstanceInterruptionBehavior": "terminate"
        }
    ]
}
```

Get public IP address of launched instance

```
instance_id=i-123456789 # copy from above output
aws --region=us-east-1 ec2 describe-instances --instance-id=$instance_id
```

Output will be similar to the below (check notes in-line in json)

```
{   
    "Reservations": [
        {   
            "Groups": [],
            "Instances": [
                {   
                    "AmiLaunchIndex": 0,
                    "ImageId": "ami-00a208c7cdba991ea",
                    "InstanceId": "i-123456789",
                    "InstanceType": "t3.medium",
                    "KeyName": "shadi",
                    "LaunchTime": "2019-12-31T07:51:32.000Z", # <<<<<<<< Make sure this matches with the current `date -u` to make sure it's a "fresh" instance.
                                                              # Forgetting to update the `client-token` field above could return an already-existing instance
                    "Monitoring": {
                        "State": "disabled"
                    },
                    "Placement": {
                        "AvailabilityZone": "us-east-1a", # <<<<<<<<<<<<<< in case we forgot the region, here it is again
                        "GroupName": "",
                        "Tenancy": "default"
                    },
                    "PrivateDnsName": "ip-111-222-333-444.ec2.internal",
                    "PrivateIpAddress": "111.222.333.444",
                    "ProductCodes": [],
                    "PublicDnsName": "ec2-111-222-333-444.compute-1.amazonaws.com", # <<<<<<<<<<<<<< useful for ssh
                    "PublicIpAddress": "111.222.333.444", # <<<<<<<<<<<<< also useful for ssh
                    "State": {
                        "Code": 16,
                        "Name": "running"
                    },
                    "StateTransitionReason": "",
                    ...
                }
            ],
            "OwnerId": "974668457921", # <<<<<<<< This is the AWS ID for autofitcloud.com
            "RequesterId": "123456789",
            "ReservationId": "r-123456789"
        }
    ]
}
```

SSH into machine

```
ssh -i /path/to/key.pem ubuntu@public-ip.amazonaws.com
```

Finally, to make sure that you're not paying for idle machines
running in your AWS account, consider using [isitfit](https://isitfit.autofitcloud.com/)
to detect idle machines in all AWS regions:

```
pip3 install isitfit
isitfit cost analyze
```

Output similar to

```
Cost-Weighted Average Utilization (CWAU) of the AWS account (EC2, Redshift):

Service Field
ec2     Start date          2019-12-24  2019-12-25  2019-12-26  2019-12-27  2019-12-28  2019-12-29  2019-12-30     2019-12-31
        End date            2019-12-24  2019-12-25  2019-12-26  2019-12-27  2019-12-28  2019-12-29  2019-12-30     2019-12-31
        Regions                      0           0           0           0           0           0           0  1 (us-east-1)
        Resources analyzed           0           0           0           0           0           0           0              2
        Billed cost                0 $         0 $         0 $         0 $         0 $         0 $         0 $            0 $
        Used cost                  0 $         0 $         0 $         0 $         0 $         0 $         0 $            0 $
        CWAU (Used/Billed)         0 %         0 %         0 %         0 %         0 %         0 %         0 %            0 %
```

and

```
isitfit cost optimize
```

with output similar to 

```
+-----------+-----------+---------------------+------------------+------------------+--------------------+-------------------------+-----------+---------------------+-----------+--------+-------------------------+
| service   | region    | resource_id         | resource_size1   | resource_size2   | classification_1   | classification_2        |   cost_3m | recommended_size1   |   savings | tags   | dt_detected             |
|-----------+-----------+---------------------+------------------+------------------+--------------------+-------------------------+-----------+---------------------+-----------+--------+-------------------------|
| EC2       | us-east-1 | i-1                 | t3.medium        |                  | Idle               | No memory metrics       |       106 | t3.small            |       -53 |        | 2019-12-31 08:21:03.... |
| EC2       | us-east-1 | i-2                 | t3.medium        |                  | Not enough data    | 1 day(s) available. ... |       106 |                     |         0 |        | 2019-12-31 08:21:03.... |
+-----------+-----------+---------------------+------------------+------------------+--------------------+-------------------------+-----------+---------------------+-----------+--------+-------------------------+
```


## Note 1: AMI ID

Change AMI ID in `launch_specification.json` as needed.

Example lists: https://cloud-images.ubuntu.com/locator/ec2/

```
  "ImageId": "ami-0bbe6b35405ecebdb"   <<< Ubuntu 18.04 in us-west-2
  ami-0400a1104d5b9caa1  << ubuntu 18.04 in us-east-1
```

Also
https://docs.aws.amazon.com/dlami/latest/devguide/deep-learning-containers-images.html
for ec2 images for deep learning.


## Note 2: ssh into port 22

If it is intended to ssh into the machine, check that the `SecurityGroupIds` field in the json file leaves port 22 open (or other ports as needed).
If this is left to be `[]`, this will default to the security group named `default`.
To check that it gives access to port 22 using awscli:

```
aws --region=us-east-1 ec2 describe-security-groups --filters Name=group-name,Values=default --filters Name=ip-permission.from-port,Values=22
```


## Note 3: tagging

I couldn't figure out how to automatically tag the instances from the command-line.
Initially I thought maybe the `launch_specification.json` supported a `Tags` key like this:

```
  "Tags": [
    {"Key": "app", "Value": "autofitcloud-howto-aws-spot-and-email"}
  ]
```

but aparently that isn't supported.


## Security

Copying your AWS access key ID and secret to the new instance might not be suitable for your use case.
You might want to create a role dedicated to the new instance,
and give the role permission to send an email with AWS SES.

Also, the `user_data.sh` file will be copied into `/var/lib/cloud/instances/` on the launched machine.
As this file contains the AWS access key ID and secret, you might want to delete it (as well as the `/root/.aws/credentials` file... yes, `/root`... I hear you)
before sharing it with other users. [reference](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)


## Other useful links

https://temp-mail.org/en/

- for temporary email


https://github.com/AutoSpotting/AutoSpotting

- works with autoscaling groups


https://github.com/vithursant/terraform-aws-spotgpu

- goes through terraform


https://github.com/apls777/spotty

- for deep learning projects
- creates an s3 bucket, main cloudformation stack, cloudformation stack per "project", snapshots of disks upon spot termination


https://github.com/Jakobovski/aws-spot-bot

- deprecated


http://technologist.pro/devops/aws-ec2-automation-using-aws-cli-and-user-data

- useful tutorial on awscli and create/terminate ec2
- also stresses the importance of tagging ... agreed
- a few years down the line, you'll have that one machine that nobody knows who it belongs to without ssh'ing into it and dissecting the filesystem
