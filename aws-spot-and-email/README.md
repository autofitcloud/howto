# howto/aws-spot-and-email

*How to request an AWS spot instance and send email upon fulfillment*

Useful for spot requests that take some time to get fulfilled

Triggered from [this reddit discussion](https://www.reddit.com/r/aws/comments/ehm1ah/best_aws_solution_for_high_memory_usage/fckm1g5)


## Usage

These are steps that will create a `t3.medium` spot instance request in `us-east-1` and send an email to the configured email address upon fulfillment.

Download this repository

```
wget ...
```

Copy templated files

```
cd howto/aws-spot-and-email
cp user_data.sh.template user_data.sh
cp launch_specification.json.template launch_specification.json
```

Replace variables in curly braces in new files with your own values, eg

```
launch_specification.json:
  KeyName: shadi

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
base64 user_data.sh > user_data.sh.b64
```

Request spot instance and have it send email on fulfillment (remember to use a new `client-token` for different requests)

```
aws --region=us-east-1 \
  ec2 request-spot-instances \
  --block-duration-minutes=60 \
  --client-token=test-isitfit-spot-6 \
  --instance-count=1 \
  --spot-price=0.3 \
  --launch-specification file://launch_specification.json
```


Check status

```
aws --region=us-east-1 ec2 describe-spot-instance-requests --spot-instance-request-ids=$req_id
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


