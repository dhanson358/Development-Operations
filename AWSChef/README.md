<pre>
  _                 _        ___ _               _   
 | |   ___ _ _  ___| |_  _  | _ \ |__ _ _ _  ___| |_ 
 | |__/ _ \ ' \/ -_) | || | |  _/ / _` | ' \/ -_)  _|
 |____\___/_||_\___|_|\_, | |_| |_\__,_|_||_\___|\__|
                      |__/                           
Infrastructure as code.
</pre>

Summary
=======

This is a modified README.md from a private repository that forms the groundwork of an AWS hosted environment managed by Chef.

Platform and management
-----------------------

The infrastructure is hosted on AWS and we use Chef for server orchestration.

Server template
---------------

The server template is based on a current Ubuntu 11.10 Cloud Guest AMI as of 20120312.
AMI's for each zone and region can be found here:

* http://cloud.ubuntu.com/ami/

You'll want a oneiric amd64 AMI that is EBS backed.

Additions to the image that support the bootstrap and orchestration process:

```
Chef
ruby-1.9.3p0
Some basic configuration. Chef should handle as much as possible.
```

Configuration of management toolset
-----------------------------------

1) The Amazon EC2 API Tools are essential:

* http://aws.amazon.com/developertools/351

1a) To work with CloudWatch (monitoring) and AutoScaling, you'll need these tools:

* http://aws.amazon.com/developertools/2534

* http://aws.amazon.com/developertools/2535

2) Export relevant environment variables and obtain certificates and keys from the AWS console. Set these in your .bash_profile:

```bash
export EC2_URL=https://ec2.us-east-1.amazonaws.com

export EC2_PRIVATE_KEY=pk-admin.pem

export EC2_CERT=cert-admin.pem

export JAVA_HOME=/System/Library/Frameworks/JavaVM.framework/Home/

export EC2_HOME=/Users/mark/ec2

export AWS_AUTO_SCALING_HOME=/Users/mark/ec2/as

export AWS_CLOUDWATCH_HOME=/Users/mark/ec2/cw

export PATH=$PATH:$EC2_HOME/bin:$AWS_AUTO_SCALING_HOME/bin:$AWS_CLOUDWATCH_HOME/bin
```

3) You'll also need the default SSH key pair added to ssh-agent:

```bash
ssh-add ~/.ssh/admin.pem
```

Bootstrap process (single server) 
---------------------------------

1) Be sure to have a functioning cloud-init user-data file:

```bash
./write-mime-multipart 10-banner 20-boot 30-user 40-ruby 50-repo 99-default-role > data
gzip data
mv data.gz $EC2_HOME
```

write-mime-multipart allows you to pass cloud-init configuration and shell scripts as one data file to the Amazon API's.

**The file 99-default-role is very important. This is the JSON passed to chef-solo.**

Example:

```bash
#!/bin/bash
#Chef run list
cat << "EOF" > /etc/chef/node.json
{
    "run_list": [ 
        "role[default]"
    ]
}
EOF
chef-solo && echo "Server ready." | wall
```

On boot, the cloud-init user-data file is passed to the server for execution. Amongst other things, it prepares the server to execute Chef recipes that are cloned from another Git repository:

* https://github.com/secret/chef

This method means that at boot time, each server has the latest Chef cookbooks available. It also encourages servers to be regularly destroyed and recreated.

2) Launch an instance with an AMI for that zone and referencing the SSH key pair:

```bash
ec2-run-instances -t m1.small ami-baba68d3 -f data.gz -k admin
```

3) Deploy code!

Bootstrap process (AutoScaling)
-------------------------------

1) Use the same data.gz you would for a single server.

2) Create a scaling config and launch one instance:

```bash
as-create-launch-config nginxscale --instance-type m1.small --image-id ami-baba68d3 --user-data-file ../data.gz --key admin
```

3) Create a scaling group. This defines the lower and upper number of servers allowed in the group:

```bash
as-create-auto-scaling-group nginxgroup --availability-zones us-east-1a --launch-configuration nginxscale --min-size 1 --max-size 5 --default-cooldown 300
```

4) Create a scaling policy. This is the action taken when a condition is met:

Create one server:

```bash
as-put-scaling-policy HighCPUPolicy --auto-scaling-group nginxgroup  --adjustment=1 --type ChangeInCapacity --cooldown 300
```

Output:

```
arn:aws:autoscaling:us-east-1:049718304780:scalingPolicy:a30c5e71-1536-4163-bc7a-df19237fd8d2:autoScalingGroupName/nginxgroup:policyName/HighCPUPolicy       
```

Terminate one server:

```bash
as-put-scaling-policy LowCPUPolicy --auto-scaling-group nginxgroup  --adjustment=1 --type ChangeInCapacity --cooldown 300                                
```

Output:

```
arn:aws:autoscaling:us-east-1:049718304780:scalingPolicy:fb162fd8-472b-420b-84f1-a45f850ad3cf:autoScalingGroupName/nginxgroup:policyName/LowCPUPolicy              
```

5) Define CloudWatch alarms that when triggered, calls an event from above:

This evaluates the CPUUtilization metric within CloudWatch over a 4 minute average to see if it exceeds 85% and adds a server:

```bash
mon-put-metric-alarm HighCPUAlarm  --comparison-operator  GreaterThanThreshold  --evaluation-periods  4 --metric-name  CPUUtilization  --namespace  "AWS/EC2"  --period  60  --statistic Average --threshold  85 --alarm-actions arn:aws:autoscaling:us-east-1:049718304780:scalingPolicy:a30c5e71-1536-4163-bc7a-df19237fd8d2:autoScalingGroupName/nginxgroup:policyName/HighCPUPolicy --dimensions "AutoScalingGroupName=nginxgroup"
```

This evaluates the CPUUtilization metric over the same time period and if it is below 40% a server is removed:

```bash
mon-put-metric-alarm LowCPUAlarm  --comparison-operator  GreaterThanThreshold  --evaluation-periods  4 --metric-name  CPUUtilization  --namespace  "AWS/EC2"  --period  60  --statistic Average --threshold  40 --alarm-actions arn:aws:autoscaling:us-east-1:049718304780:scalingPolicy:a30c5e71-1536-4163-bc7a-df19237fd8d2:autoScalingGroupName/nginxgroup:policyName/HighCPUPolicy --dimensions "AutoScalingGroupName=nginxgroup"
```

**It's important to note that Amazon doesn't offer a native method of monitoring memory usage. You can however define your own metrics to trigger actions. This lends itself nicely to application metric based scaling.**

Another useful toolset is the AWS Missing Tools:

* http://code.google.com/p/aws-missing-tools/
