---
layout: post
title: S3 Object Expiration
cdn: aws-object-expiration
---

**Situation**: You're putting a bunch of files into S3 and don't want it to grow uncontrollably.  Maybe your build process is generating a bunch of zip files that are being used with CodeDeploy.  If you're not careful you're gonna have a lot of files sitting around.

You could fix this with a cronjob to cleanup the S3 bucket, but there's an even easier fix.  Use object expiration!  This isn't a new feature (it was [announced back in 2011](https://aws.amazon.com/blogs/aws/amazon-s3-object-expiration/)), but it might be something that you haven't setup.

## Getting Started

Full documentation is located [here](http://docs.aws.amazon.com/AmazonS3/latest/dev/manage-lifecycle-using-console.html).  You can set this up using either the console or the AWS CLI.  I'll be using the [put-bucket-lifecycle](http://docs.aws.amazon.com/cli/latest/reference/s3api/put-bucket-lifecycle.html) option of the CLI.

### Example 1: Delete Objects After 5 days

For lifecycle management the word "expire" really means delete.  That policy could be placed in `lifecycle.json`:

{% highlight json %}
{
  "Rules": [
    {
      "ID": "Delete objects older than 5 days",
      "Prefix": "",
      "Status": "Enabled",
      "Expiration": {
        "Days": 5
      }
    }
  ]
}
{% endhighlight %}

You could then apply to the bucket like this:

    aws --profile=<NOT NECESSARY IF YOU WANT THE DEFAULT> --region=<AWS_REGION> s3api put-bucket-lifecycle --bucket <BUCKET_NAME> --lifecycle-configuration file://lifecycle.json


### Example 2: Move objects to Glacier after 5 days, store indefinitely

If you don't specify an expiration the object will stay stored indefintely.  All you do is specify that you want to move objects to `GLACIER` after 5 days.  The policy would like this:

{% highlight json %}
{
    "Rules": [
        {
            "Status": "Enabled",
            "Prefix": "",
            "Transition": {
                "Days": 5,
                "StorageClass": "GLACIER"
            },
            "ID": "Glacier after 5 days, store indefinitely"
        }
    ]
}
{% endhighlight %}

You could then apply to the bucket like this:

    aws --profile=<NOT NECESSARY IF YOU WANT THE DEFAULT> --region=<AWS_REGION> s3api put-bucket-lifecycle --bucket <BUCKET_NAME> --lifecycle-configuration file://lifecycle.json

### Example 3: Move Objects to Glacier after 5 days, delete after 30 days

You can also specify 2 rules.  The first rule moves the object to `GLACIER` after 5 days.  After 30 days the object will expire from `GLACIER`.  The policy looks like this:

{% highlight json %}
{
    "Rules": [
        {
            "Status": "Enabled",
            "Prefix": "",
            "Transition": {
                "Days": 5,
                "StorageClass": "GLACIER"
            },
            "Expiration": {
                "Days": 30
            },
            "ID": "Glacier after 5 days, expire after 30 days"
        }
    ]
}
{% endhighlight %}

You could then apply to the bucket like this:

    aws --profile=<NOT NECESSARY IF YOU WANT THE DEFAULT> --region=<AWS_REGION> s3api put-bucket-lifecycle --bucket <BUCKET_NAME> --lifecycle-configuration file://lifecycle.json

### What policies are defined for a bucket?

You can see what policies are in effect for a bucket by running this:

{% highlight bash %}
$ aws --profile=<NOT NECESSARY IF YOU WANT THE DEFAULT> s3api get-bucket-lifecycle --bucket <BUCKET_NAME> --region=<AWS_REGION>
{
    "Rules": [
        {
            "Status": "Enabled",
            "Prefix": null,
            "Expiration": {
                "Days": 5
            },
            "ID": "Delete objects older than 5 days"
        }
    ]
}
{% endhighlight %}

That's it.  Once the policy is in place you don't have any additional work to do.  Happy expiration!