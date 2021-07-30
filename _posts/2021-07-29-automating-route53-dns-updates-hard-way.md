---
title: Automating Route53 DNS Updates for Blue/Green Deployments (the hard way)
description: How to automate Route53 DNS Updates for Blue/Green Deployments (the hard way)
author: James Ingold
published: true
---

#### Blue/Green Architecture

A blue/green architecture in some form or fashion is a necessity for highly available systems. This means that you have at least two seperate environments. The blue one is in use, handling live traffic while the green one is a clone of the production one and ready to go in case of emergency or routine maintenance. Ideally, these systems are in different regions and possibly even different cloud providers depending on your need to reduce risk. There are several ways to achieve blue/green deployments and it could be as simple as having two different servers which you can switch between. Regardless of your setup, at some point you will have to switch the DNS to point from the blue environment to the green environment. For example, [AWS CodePipeline](https://aws.amazon.com/quickstart/architecture/blue-green-deployment/) manages this for Elastic Beanstalk. Most environments won't manage the DNS updates for you though, so in this post I'll show you how to script out updates to AWS Route53 to manage blue/green DNS the hard way.

As a side note, it's good to practice switching environments so that the first time isn't under the intense pressure of something going wrong. Switching which environment is live should be as seamingless as possible, with no downtime or stress. In this example, I'll be using Route53 to switch DNS urls from one AWS elastic load balancer to another with a python script.

Boto, the python library for AWS, isn't the easiest to use for Route53. I'll break down each step of the script but if you want to skip ahead, [here's the full gist.](https://gist.github.com/james-ingold/5458f137b95a34aff4f5676daec23acd) We're going to build a python utility script to handle updating DNS entries and then a shell script for handling bulk updates. Let's write some code!

##### Python Script for Updating Route53 Entries

Create new python file and add the following code:

```python
#!/usr/bin/env python

"""
Update DNS for a RecordSet on AWS Route53
"""
import argparse
import sys

import boto3

parser = argparse.ArgumentParser()

parser.add_argument("--zoneId", type=str, default="", help="Route53 Zone Id, recommend setting this as an environment variable")
parser.add_argument("--hostname", type=str, default="", help="The hostname to update example.domain.com")
parser.add_argument("--dns", type=str, default="", help="The new dns value")

args = parser.parse_args()

route53 = boto3.client('route53')
zoneid = args.zoneId
hostname = args.hostname
CNAME = args.dns


if not zoneid or not hostname or not CNAME:
   print("Please provide a zone id, hostname and new dns")
   return
```

We're using argparse to pass in our zoneId, hostname, and the new dns. I recommend setting your hosted zone id as an environment variable so you don't have to remember it. If you only have one Route53 hosted zone, you can use Boto to get all the hosted zones (route53.get_all_hosted_zones()) and then use the first one.

Now that we've got our AWS Route53 Hosted Zone ID, we can write the update DNS function. I'm going to use a shell script to pass in an array of domain names because in production, there are some domains that I don't want to update. If you have a simpler set up, you could iterate through the record sets and update each DNS without the need for the shell script. An environment variable could also hold the domains to update without using the shell script as well.

```python
def updatedns(hostname, newdns):
 sets = route53.list_resource_record_sets(HostedZoneId=zoneid)

 for rset in sets['ResourceRecordSets']:
    if rset['Name'] == hostname and rset['Type'] == 'CNAME':
        curdnsrecord = rset['ResourceRecords']
        print(curdnsrecord)
        if type(curdnsrecord) in [list, tuple, set]:
            for record in curdnsrecord:
                curdns = record
        # print('Current DNS CNAME: %s' % curdns)
        curttl = rset['TTL']
        # print('Current DNS TTL: %s' % curttl)

        if curdns != newdns:
            # UPSERT the record
            print('Updating %s' % hostname)
            route53.change_resource_record_sets(
              HostedZoneId=zoneid,
              ChangeBatch={
                'Changes': [
                  {
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                      'Name': hostname,
                      'Type': 'CNAME',
                      'TTL': curttl,
                      'ResourceRecords': [
                        {
                          'Value': newdns
                        }
                      ]
                    }
                  }
                ]
              }
            )
```

We're passing in the hostname to update and the new DNS entry to the updatedns function. From there, we check out each RecordSet and see if it matches the hostname that we're trying to update. If it does, we copy over the time to live property and then change the dns if it doesn't match what was passed in. The Route53 client's change_resource_record_set is handy because we can use the 'UPSERT' action to create or update by hostname.

That's it for the python file, check out the full gist [here](https://gist.github.com/james-ingold/5458f137b95a34aff4f5676daec23acd).

##### Bonus: Shell script to Pass Domains to Update

```bash
#!bin/bash
# Shell script to update AWS Route53 DNS when switching Blue/Green Deployments

if [ $# -lt 1 ]
then
  echo "Please supply $1 destination DNS address (likely an ELB address)"
  return 1
fi

# array of domains to update
domains=("example.mydomain.com" "example2.mydomain.com")

for i in "${domains[@]}"
do
  echo "updating $i";
  python updatedns.py --zoneId="$MY_ROUTE53_HOSTED_ZONE" --hostname="$i" --dns="$1"

done

echo "Finished updating Route53 records"
```

##### The End

That's it! We've now automated changing DNS for blue/green environments using AWS Route53, the hard way. If you know any easier way, let me know! Happy Codings

##### References:

[Boto Route53 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/route53.html){:target="\_blank"}

[updatedns.py gist](https://gist.github.com/james-ingold/5458f137b95a34aff4f5676daec23acd)
