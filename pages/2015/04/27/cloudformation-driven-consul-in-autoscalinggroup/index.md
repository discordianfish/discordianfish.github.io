---
title: 'CloudFormation driven Consul in AutoScalingGroup'
date: 2015-04-27 13:22:30 +0000
tags: consul aws cloudformation
layout: post
---
<p style="font-size:10px">
<img src="/content/images/2015/Apr/dreamy_consul-2.jpg"><br/>
<span>Jessie Eastland, <a href="http://commons.wikimedia.org/wiki/File:Dreamy_Twilight.jpg">„Dreamy Twilight“</a>, Consul Logo added</span>, <a href="http://creativecommons.org/licenses/by-sa/3.0/legalcode/">CC BY-SA 3.0</a></span>
</p>

CloudFormation templates are a great way to manage AWS resources. All resources for a stack are configured in a json and CloudFormation takes care of creating or updating your stack.
Consul is a distributed, consistent data store for Service Discovery and configuration which also features health checks and supports for distributed locks.
Consul deployed and updated via CloudFormation provides a nice foundation for any kind of modern infrastructure.

# AutoScalingGroup
If you specify instances in your stack directly by using the `AWS::EC2::Instance` resource, updating anything that requires recreation of the instance will bring up a new instance and terminate the old one. If you need more control over the recreation, the instances need to be managed via a AutoScalingGroup.

By default, updating a AutoScalingGroup won't affect the instances. You need to manually terminate them and let AutoScale spin up a new instance.
To automatically update the instances, you can specify a UpdatePolicy requiring at least n instances in service while doing a rolling upgrade.

In the case of consul this isn't enough. Without checking that a new consul node actually came up and connected successfully to the cluster we might lose to quorum and the cluster becomes stale.

Fortunately CloudFormation offers a UpdatePolicy option WaitOnResourceSignals which can be used to signal that any new consul node coming up connects sucessfully to the cluster before the any new instance gets terminated.

# UpdatePolicy
The changes to the CloudFormation template are straight forward. In the AutoScalingGroup we add a `UpdatePolicy` with `WaitOnResourceSignals: true` like this:

```
"UpdatePolicy" : {
  "AutoScalingRollingUpdate" : {
    "MinInstancesInService" : 2,
    "PauseTime" : "PT15M",
    "WaitOnResourceSignals" : true
  }
}
```

If `WaitOnResourceSignals` is set,

> AWS CloudFormation suspends the update of an Auto Scaling
> group after any new Amazon EC2 instances are launched into
> the group. AWS CloudFormation must receive a signal from
> each new instance within the specified pause time before
> AWS CloudFormation continues the update.

Now we simply need to make sure any new instance sends a success signal after it came up and consul connected successfully to the cluster.

# Send success signal
Once a new instance starts, we need to wait for consul to join the cluster and replicate. Since we need to guarantee that a new node actually joined the cluster, not some partition of it, we need something that goes through the raft log.
When looking at the consul documentation, there are a few endpoint we could use. Unfortunately the documentation doesn't state what the consistency guarantees of status endpoints like `peers` are, so [I had to ask](https://github.com/hashicorp/consul/issues/880). Turns out, we can trust `peers`.
With that knowing, we can simply wait for consul being ready as part of the UserData script, then signal success by using the `cfn-signal` tool:

```
"UserData": { "Fn::Base64" : { "Fn::Join" : ["", [
  "IP=$(ip addr show dev eth0|awk '/inet /{print $2}'|cut -d/ -f1)\n",
  "while ! curl -s http://localhost:8500/v1/status/peers | grep -q $IP:; do echo Waiting for consul; sleep 1; done\n",
  "cfn-signal --resource InfraScalingGroup --stack ", {"Ref": "AWS::StackName"}, " --region ", {"Ref" : "AWS::Region"}, "\n"
}
```

# AMI
The ami runs consul on start and uses a IAM role with read-only EC2 access. Since we tagged all consul instances, this allows the init script to discover peers via the AWS API:

```
URL="http://169.254.169.254/latest/"
ID=$(curl $URL/meta-data/instance-id)
REGION=$(curl $URL/dynamic/instance-identity/document | \
  jq -r .region)

SERVERS=$(aws --region $REGION ec2 describe-instances \
  --filters \
  "Name=tag:aws:cloudformation:stack-id,Values=$STACK_ID" \
  "Name=tag:role,Values=consul" \
  "Name=instance-state-name,Values=running" | \
    jq -r '.Reservations[].Instances[].PrivateIpAddress' \
)
```

We also need to check that SERVERS include the necessary number of peers to form a cluster. Since we're using runit which restarts failing jobs, we just fail if we found less peers than expected and wait to get restarted:

```
if [ $(echo "$SERVERS" | wc -l) -lt $BOOTSTRAP_EXPECT ]
then
  echo "Not enough peers, expected $BOOTSTRAP_EXPECT nodes but got $SERVERS"
  exit 1
fi
```

Once all peers are up and running, we can start consul:
```
exec /usr/bin/consul agent -data-dir /var/lib/consul \
  -config-dir=/etc/consul \
  $(echo "$SERVERS" | sed 's/^/ -retry-join /' | tr -d '\n')
```

# Updating the Stack
With all this in place you can do a fully automated rolling upgrade while keeping a quorum. Just update the stack and you should get something like this while having a fully operable cluster:

![Update Stack Log](/content/images/2015/Apr/output_710_280.gif)
