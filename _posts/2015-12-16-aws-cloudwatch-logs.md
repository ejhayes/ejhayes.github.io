---
layout: post
title: Log aggregation with CloudWatch Logs
cdn: aws-object-expiration
---

Log aggregation is now easier than ever to setup thanks to [CloudWatch Logs](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/WhatIsCloudWatchLogs.html).  If you aren't familiar with log management, check out this [article](http://www.csoonline.com/article/2126060/network-security/log-management-basics.html) for a brief introduction.

I'm a fan of CloudWatch Logs for several reasons:

* Easy to setup
* Integrates in the AWS Ecosystem
* [Retention policies](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/SettingLogRetention.html)
* Able to [filter log data](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/Counting404Responses.html) to trigger alerts or other [custom behavior](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/Subscriptions.html)

This article highlights another cool feature--you can tail your logs in near realtime too!  So if you're a fan of `tail -f /path/to/mylog.log`, this article is for you.

## Create Policies for CloudWatch Logs

If you're running this from EC2, you can add the following policy to the IAM Role associated with the instance:

    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "logs:CreateLogGroup",
            "logs:CreateLogStream",
            "logs:PutLogEvents",
            "logs:DescribeLogStreams"
        ],
          "Resource": [
            "arn:aws:logs:*:*:*"
        ]
      }
     ]
    }

## Install CloudWatch Logs Agent

You can install the agent like this (I'm using Ubuntu 14.04 on EC2):

    wget https://s3.amazonaws.com/aws-cloudwatch/downloads/latest/awslogs-agent-setup.py
    sudo python ./awslogs-agent-setup.py --region us-east-1

The installer will configure log monitoring for `syslog` by default.  I find it easier to edit the file manually rather than through the tool.  For our example, lets say that we want to send the following log files to CloudWatch:

* `/var/log/nginx/error.log`
* `/var/log/nginx/access.log`

Just add the following to the bottom of `/var/awslogs/etc/awslogs.conf` (replace `APP_ID` with a more meaningful name, like `acme-api` or `acme-web`):

    [/var/log/nginx/error.log]
    datetime_format = %Y-%m-%d %H:%M:%S
    file = /var/log/nginx/error.log
    buffer_duration = 5000
    log_stream_name = APP_ID {instance_id}
    initial_position = end_of_file
    log_group_name = /var/log/nginx/error.log

    [/var/log/nginx/access.log]
    datetime_format = %Y-%m-%d %H:%M:%S
    file = /var/log/nginx/access.log
    buffer_duration = 5000
    log_stream_name = APP_ID {instance_id}
    initial_position = end_of_file
    log_group_name = /var/log/nginx/access.log

Then restart the logs service by running `sudo service awslogs restart`.

## Get your log data

To get log events you can use the AWS CLI like this:

    $ aws logs --profile=my_profile get-log-events --log-group-name /var/log/nginx/error.log --log-stream-name "acme-web i-07fdf4d3"
    {
        "nextForwardToken": "f/32258355170809589614344124816922785120512659126568615936",
        "events": [
            {
                "ingestionTime": 1446513625731,
                "timestamp": 1446513629293,
                "message": "App 22007 stdout: { specialtyData: "
            },
    ...

I prefer to use the human friendly [awslogs](https://github.com/jorgebastida/awslogs) tool (`pip install awslogs`) so that I can tail logs like this:

```bash
$ AWS_PROFILE=my_profile awslogs get /var/log/nginx/error.log "acme-web*" --watch
/var/log/nginx/error.log acme-web i-07fdf4d3 App 12432 stdout:
/var/log/nginx/error.log acme-web i-07fdf4d3 App 12432 stdout: Node Environment is: production
/var/log/nginx/error.log acme-web i-07fdf4d3 App 12432 stdout: Creating httpServer
/var/log/nginx/error.log acme-web i-07fdf4d3 App 12432 stdout: listening on 80
...
```

**NOTE**: Ensure that you're using version `0.1.2` or higher of this tool.  There was an issue with `--watch` that caused the command to hang.

This just scratches the surface of what you can do.  Happy logging!
