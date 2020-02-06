# howto/02-aws-spot-persistent-disk

*How to use aws spot instances with a persistent disk*

Useful for running a minecraft server on cheaper spot instances and persistent the game state.

This tutorial was written in response to [this reddit discussion](https://www.reddit.com/r/aws/comments/dmmr1x/i_have_a_question_about_burstable_cpus_in_ec2/fg2x8ds/)


## Usage

The pre-requisite for this tutorial is an AWS EBS disk containing the following

- minecraft server files: `eula.txt, server.properties, world, ...`
- systemd service that launches minecraft on boot: `minecraft.service`


In the rest of this tutorial, I will refer to this disk as "minecraft disk"


The following steps will

- Request a `c5.large` spot instance in `us-east-1`
- Attach the minecraft disk
- Start the minecraft server software
- Terminate the instance after 60 minutes
- Installing the minecraft server is out-of-scope for this tutorial


Download the repository

```
git clone https://github.com/autofitcloud/howto/ autofitcloud-howto
cd autofitcloud-howto/02-aws-spot-persistent-disk

OR IF YOU DONT HAVE GIT

wget https://github.com/autofitcloud/howto/archive/master.zip -O autofitcloud_howto.zip
unzip autofitcloud_howto.zip
mv howto-master autofitcloud-howto
cd autofitcloud-howto/02-aws-spot-persistent-disk
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
  AvailabilityZone # use the same zone as the minecraft disk, otherwise aws will not let you attach it

user_data.sh:
  aws_access_key_id: abcdef # copy from ~/.aws/credentials
  aws_secret_access_key: abcdef # ditto
  volume_id: vol-12345 # AWS volume ID of the minecraft disk
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

Request spot instance and have it auto-attach the minecraft disk (remember to use a new `client-token` for different requests)

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

Check status of spot request

```
req_id=sir-12345 # copy this from output of above `ec2 request-spot-instances` command, field value of `SpotInstanceRequestId`
aws --region=us-east-1 ec2 describe-spot-instance-requests --spot-instance-request-ids=$req_id
```

Get public IP address of launched instance

```
instance_id=i-123456789 # copy from above output
aws --region=us-east-1 ec2 describe-instances --instance-id=$instance_id
```

Log into the new IP from minecraft and check that indeed your constructed world is still maintained.
