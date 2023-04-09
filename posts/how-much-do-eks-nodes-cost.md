---
title: How much do my Amazon EKS nodes cost?
date: 2022-02-05
tags:
- amazon-eks
- Kubernetes
layout: layouts/post.njk
---

If you want to know the cost of running Amazon EKS nodes, AWS Cost Explorer is your friend.

[AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/) helps you analyze your AWS bill. Cost Explorer has an easy-to-use interface that lets you visualize, understand, and manage your AWS costs and usage over time.

Cost Explorer supports breaking down costs using [tags](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html). A tag is a label that you assign to an AWS resource. Tags enable you to categorize your AWS resources by, for example, purpose, owner, or environment.

Kubernetes cluster deployment tools like `eksctl` automatically add default tags to EC2 instances that are part of a managed node group. Customer can view tags attached to their nodes in the AWS Management Console.

To view the tags attached to your EKS nodes, go to the AWS Management Console and inspect the tags. You can also use the [`DescribeTags` API](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeTags.html) to query tags using AWS CLI or SDK.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/99997738-88A8-4420-9881-286553E32476_2/hqNzdOTLsvdLYOyrPn70lF2cToHkSbhIDtm4xr8RA08z/Image.jpeg)

Pick an EC2 instance that's part of your EKS cluster, note its tags, select a tag, like `eks:cluster-name`, and verify that all nodes have that label. You will later use this tag to separate the costs of Kubernetes nodes from the rest of EC2 usage in your account(s).

## Activate tags in billing

Once you have identified the tag using which you are going to identify EKS EC2 nodes, you’ll have to activate that tag in the AWS Billing and Cost Management console. You’ll need [privileges](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/control-access-billing.html) to make changes.

To activate your tags using the AWS Management Console:

1. Sign in to the AWS Management Console and open the AWS Billing console at [https://console.aws.amazon.com/billing/](https://console.aws.amazon.com/billing/).
2. In the navigation pane, choose Cost Allocation Tags.
3. Select the tags that you want to activate.
4. Choose Activate.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/263EE126-DAD2-4F52-856E-6F065BF89E53_2/AwhushwYYUHGexRRGMqTseZBAfi0u2rQP8k2k8xeNpoz/Image.jpeg)

After you select your tags for activation, it can take **up to 24 hours** for tags to activate.

Billing has AWS-generated cost allocation tags that customers can use to identify costs.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/0C3F9990-AB32-4876-85EB-CE7F7D9E4C83/123DF0AF-8C9D-4625-8816-FF6A2511B596_2/lHrGroWK3tRZFJVuLqOdx2xPWKEEADFIo5teWiGeIbgz/Image.jpeg)

## View charges in AWS Cost Explorer

Navigate to AWS Cost Explorer and create a new report

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/3B13ED5A-B51C-4109-8D51-E3BC9187BF8A_2/dv6xOgqDiX8lQMHmPz0yqeySZ6OgPqEBAqRLyEifu3gz/Image.jpeg)

In the next screen, choose **Cost and usage**. Then scroll to the bottom of page and choose **create report.**

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/0C3F9990-AB32-4876-85EB-CE7F7D9E4C83/75A8974D-92E5-487B-8A66-F1DA634E81B0_2/aoYH9yyFtz8vHHrxMuthXXXaBlGYAuLdeUxbPeTqDCIz/Image.jpeg)

In the next screen, configure a **Service** filter, select EC2-Instances, and apply filter.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/0C3F9990-AB32-4876-85EB-CE7F7D9E4C83/A360630B-8E50-42EE-8245-C5AADFA5BFFC_2/wmtRAMwgAJWxXCTAMjeTGPogUhEmOdiBnRxOuTylyuYz/Image.jpeg)

Create a **Tag** filter, select the tag you activated in billing, and apply. Be sure to select only `true`.

In my case, my nodes had the `eks:cluster-name` tag.

![Image](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/7B8C4159-87D5-4FBF-847F-0AA3DB34730C/22D7482D-E383-43BC-B123-5D4C805D7C52_2/PPl1BfJ7puxg0H4lPCLqyxpX58r4cEAO0R7iTzLRsLIz/Image)

Now Cost Explorer shows just the charges for my cluster.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/AE2DDB1D-DD37-409F-81C6-B5F447210571_2/YpYzKtND10nuEyoY0qifZWzkxyyX31CW511EIjI02xwz/Image.jpeg)

Now I know that in Feb ’22, I paid roughly $3000 for the nodes that are part of the cluster named “Socrates.”

