---
layout: post
title: "Building a cheap email forwarding service with Gmail and AWS CDK"
date: 2020-07-01
categories: ["aws", "consulting", "aws-cdk"]
---

TODO: This blog post is entirely outdated, as I no longer use this stack. I use Zoho as a very cheap $1/month mailbox and SMTP server, and forward work emails to my personal ["Hey"][hey] account.

--

When you register a domain, you don't get email for free. One of the first decisions I had to make after purchasing the domain for this blog was where to host my email. Initially I went with [ProtonMail][protonmail], lured by relative low cost for a single custom domain and feeling paranoid about security.

After using ProtonMail for two years, I have only one thing to say about it: The webapp is slow and clumsy, and the encryption is pointless unless the recipient is also using ProtonMail. (This is always the downfall of encrypted messaging services.) They do offer POP and IMAP through a bridge, but it's not as though Apple Mail is a killer email reading experience either. I still prefer Gmail, in spite of the privacy and monopoly issues.

I demoed and priced out other services, but didn't find anything that great except ["Hey"][hey].

## Brainwave

I thought about it for a few minutes, and then it hit me: **I can use Amazon SES to send and receive email, and forward it to my free Gmail account.**

I have been learning the AWS CDK, and absolutely loving it. Let's begin!

I do not think this is violating any terms of service, but please [let me know](mailto:info@willmakley.dev) if you disagree.

## 1. Start a new [AWS CDK][cdk] project.

I used typescript, but you can use whatever language you prefer.

```shell
# Install the CDK
npm -g install aws-cdk

# Make new project
mkdir email-forwarder
cd email-forwarder
cdk init -l typescript

# Start the Typescript automatic compiler
npm run watch
```

If you have never used the CDK before, you will probably need to configure some AWS CLI credentials, and as well as bootstrap CDK assets in your region.

1. Install the [AWS CLI][aws-cli].
2. Create an IAM admin user to deploy your stack (outside the scope of this tutorial).
3. Run `aws configure` and set your API credentials and region.
4. Run `cdk bootstrap` to deploy some resources to the region that the CDK will use.

If any of this fails, please refer to Amazon's documentation for installing the CLI and CDK!

## 2. Install Dependencies

```shell
npm i -S \
  @aws-cdk/aws-ec2 \
  @aws-cdk/aws-iam \
  @aws-cdk/aws-lambda \
  @aws-cdk/aws-lambda-destinations \
  @aws-cdk/aws-route53 \
  @aws-cdk/aws-route53-targets \
  @aws-cdk/aws-s3 \
  @aws-cdk/aws-ses \
  @aws-cdk/aws-ses-actions \
  @aws-cdk/aws-sns \
  @aws-cdk/aws-sns-subscriptions
```

## 3. Set up CDK with knowledge of your account and preferred region.

For this stack to work, you will likely need to hardcode the region and account to deploy it to. The CDK needs this information to perform lookups and other tasks.

1. Edit 'bin/email-forwarder.ts'. It should look like this:

```typescript
const app = new cdk.App();
new EmailForwarderStack(app, 'SharedResourcesStack');
```

Make the following changes:

```typescript
const app = new cdk.App();
new EmailForwarderStack(app, 'SharedResourcesStack', {
  env: {
    region: "your-region",
    account: "your-numerical-account-id" // <- without dashes
  }
});
```
If you don't like committing your account ID to version control, you can use an environment variable:

```typescript
const app = new cdk.App();
new EmailForwarderStack(app, 'SharedResourcesStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT, 
    region: process.env.CDK_DEFAULT_REGION 
  }
});
```
See [https://docs.aws.amazon.com/cdk/latest/guide/environments.html](https://docs.aws.amazon.com/cdk/latest/guide/environments.html) for more info.


## 4. Define a hosted zone for your domain using the CDK

Edit **lib/email-forwarder-stack.ts**

```typescript
import * as cdk from '@aws-cdk/core';
import * as r53 from '@aws-cdk/aws-route53';

export class EmailForwarderStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const zone = new r53.PublicHostedZone(this, 'MyDomain', {
      zoneName: 'yourdomainhere'
    });
  }
}

```

```shell
cdk deploy
```

## 5. Add your Domain to SES

![Verifying a Domain](/assets/img/verifying-a-domain-1.png)

1. Open your AWS console.
2. Navigate to "SES" (Simple Email Service).
3. Switch to your region.
4. Click "Verify a New Domain".
5. Enter your domain name, and check the box "Generate DKIM Settings".

You will be presented with a series of DNS records to create to verify ownership of the domain.

## 5. Create Verification DNS Records

Back in email-forwarder-stack.ts, let's add these records to our zone:

```typescript
const zone = new r53.PublicHostedZone(this, 'MyDomain', {
  zoneName: 'yourdomainhere'
});
new r53.TxtRecord(this, 'SESVerification', {
  zone: zone,
  recordName: '_amazonses',
  values: ['COPY FROM CONSOLE']
});
new r53.CnameRecord(this, 'DKIM1', {
  zone: zone,
  recordName: 'COPYME._domainkey',
  domainName: 'COPYME.dkim.amazonses.com.'
});
new r53.CnameRecord(this, 'DKIM2', {
  zone: zone,
  recordName: 'COPYME2._domainkey',
  domainName: 'COPYME2.dkim.amazonses.com.'
});
new r53.CnameRecord(this, 'DKIM3', {
  zone: zone,
  recordName: 'COPYME3._domainkey',
  domainName: 'COPYME3.dkim.amazonses.com.'
});
```

Terminal:

```shell
cdk deploy
```

Review the Cloudformation diff and choose "yes" to create the resources.

Soon, things should turn green:

![Verified Domains](/assets/img/ses-domain-verified.png)

(No, my personal verification TXT record is not secret! All DNS records are public! You can go query it right now if you want!)

Woohoo!

## 6. Allow SES to send and receive email on our behalf.

Create an SPF and MX record:

```typescript
new r53.TxtRecord(this, 'TxtRecords', {
  zone: zone,
  values: [
    'v=spf1 include:amazonses.com ~all',
    // Additional '@' TXT records for your domain would go here.
  ]
});

new r53.MxRecord(this, 'MX', {
  zone: zone,
  values: [
    {
      // If you know a prettier way to inject the region, let me know!
      hostName: `inbound-smtp.${cdk.Fn.ref('AWS::Region')}.amazonaws.com.`,
      priority: 10
    }
  ],
  ttl: cdk.Duration.minutes(5) // in case something goes wrong.
});
```

Note that we set the MX TTL to 5 minutes, in case we totally fubar'd something and need to send our email somewhere else quickly. Don't forget to change it to 30-60 minutes after you are sure everything is working!

## 7. Create an S3 bucket to store emails.

We are going to use an SES ruleset to store all incoming emails in an S3 bucket, and use a lambda to forward them to our personal gmail account.

Here we create a bucket that moves incoming emails to the cheapest storage after 30 days, and deletes them after 1 year. (To be honest, we could probably safely delete them after 30 days.)

```typescript
import * as s3 from '@aws-cdk/aws-s3';

// ... previous code ...

const bucket = new s3.Bucket(this, 'Emails', {
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  lifecycleRules: [
    {
      // delete after 1 year
      expiration: cdk.Duration.days(365),
      // move to cheapest storage after 30 days
      transitions: [
        {
          transitionAfter: cdk.Duration.days(30),
          storageClass: s3.StorageClass.ONE_ZONE_INFREQUENT_ACCESS,
        }
      ]
    }
  ]
});
```

## 8. Create a Lambda to forward emails stored in S3 to Gmail.

This is where things get fun. It turns out that forwarding email is complicated, and I played around with a lot of open-source code before I found a forwarder that truly did what I want.

First, let's define it in the CDK. I like to create an SNS topic to send lambda failure notifications to, but this is totally optional.

```typescript
import * as lambda from '@aws-cdk/aws-lambda';
import * as sns from '@aws-cdk/aws-sns';
import { EmailSubscription } from "@aws-cdk/aws-sns-subscriptions";
import { SnsDestination } from "@aws-cdk/aws-lambda-destinations";

// ... previous code ...

const errorTopic = new sns.Topic(this, 'ErrorTopic');
errorTopic.addSubscription(
  new EmailSubscription('your-email-here')
);

const forwarderFn = new lambda.Function(this, 'ForwarderFn', {
  runtime: lambda.Runtime.NODEJS_12_X,
  code: lambda.Code.fromAsset('lambda'),
  handler: 'index.handler',
  environment: {
    "MailS3Bucket": bucket.bucketName,
    "MailS3Prefix": "emails",
    "MailSender": `info@${zone.zoneName}`,
    "MailRecipient": "wmakley@gmail.com",
  },
  timeout: cdk.Duration.seconds(30),
  onFailure: new SnsDestination(errorTopic),
});
```


[protonmail]: https://protonmail.com/
[cdk]: https://aws.amazon.com/cdk/
[hey]: https://hey.com/
[pnpm]: https://pnpm.js.org/
[aws-cli]: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html
