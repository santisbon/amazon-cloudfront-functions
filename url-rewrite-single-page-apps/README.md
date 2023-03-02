## URL rewrite for single page applications

**CloudFront Functions event type: viewer request**

### Description

You can use this function to perform a URL rewrite e.g. to append `index.html` to the end of URLs that don't include a filename or extension; or strip everything after the domain from the URL for SPAs that use client-side routing. This is particularly useful for single page applications or statically-generated websites using frameworks like React, Angular, Vue, Gatsby, or Hugo. These sites are usually stored in an S3 bucket and served through CloudFront for caching. Typically, these applications remove the filename and extension from the URL path. For example, if a user went to `www.example.com/blog`, the actual file in S3 is stored at `<bucket-name>/blog/index.html`. In order for CloudFront to direct the request to the correct file in S3, you need to rewrite the URL to become `www.example.com/blog/index.html` before fetching the file from S3.  

There are two sample functions.  
The function in `index.js` intercepts incoming requests to CloudFront and checks that there is a filename and extension. If there isn't a filename and extension, or if the URI ends with a "/", the function appends index.html to the URI.  
The function in the CloudFormation template removes anything after the domain from the URL.

There is a feature in CloudFront called the [default root object](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DefaultRootObject.html) that allows you to specify an index document that applies to the root object only, but not on any subfolders. For example, if you set up index.html as the default root object and a user goes to `www.example.com`, CloudFront automatically rewrites the request to `www.example.com/index.html`. But if a user goes to `www.example.com/blog`, this request is no longer on the root directory, and therefore CloudFront does not rewrite this URL and instead sends it to the origin as is. This function handles rewriting URLs for the root directory and all subfolders. Therefore, you don't need to set up a default root object in CloudFront when you use this function (although there is no harm in setting it up).

**Note:** If you are using [S3 static website hosting](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html), you don't need to use this function. S3 static website hosting allows you to set up an [index document](https://docs.aws.amazon.com/AmazonS3/latest/dev/IndexDocumentSupport.html). An index document is a webpage that Amazon S3 returns when any request lacks a filename, regardless of whether it's for the root of a website or a subfolder. This Amazon S3 feature performs the same action as this function.

### Creating the function

[Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-function.html)

When you create a function the response contains an Amazon Resource Name (ARN) that uniquely identifies the function, and the function’s stage. By default, when you create it it’s in the `DEVELOPMENT` stage. In this stage, you can test the function.  

This example function rewrites the URL to eliminate anything after the domain. Useful for Single Page Apps that use client-side routing with libraries like React Router.
```shell
REGION=us-east-1
STACK=url-rewrite-stack
TEMPLATE="./url-rewrite-single-page-apps/cfn-template.yaml"
AUTOPUBLISH=false

aws cloudformation deploy \
    --region $REGION \
    --stack-name $STACK \
    --template-file $TEMPLATE \
    --parameter-overrides AutoPublishParam=$AUTOPUBLISH
```

### Testing the function

[Reference](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/test-function.html)

To validate that the function is working as expected, you can use the JSON test objects in the `test-objects` directory. To test, use the `test-function` CLI command as shown in the following example:

Get the `ETag` value of the function to test it.
```shell
aws cloudfront describe-function --name $STACK-rewriteFunction
```

```shell
ETAG=EXXXXXXXXXXXXX

aws cloudfront test-function \
    --name $STACK-rewriteFunction \
    --if-match $ETAG \
    --event-object fileb://url-rewrite-single-page-apps/test-objects/file-name-and-extension.json \
    --stage DEVELOPMENT
```

If the function has been set up correctly, you should see the `uri` being updated to `index.html` (or whatever change you've made to the request) in the `FunctionOutput` JSON object:
```
{
    "TestResult": {
        "FunctionSummary": {
            "Name": "url-rewrite-single-page-apps",
            "Status": "UNPUBLISHED",
            "FunctionConfig": {
                "Comment": "",
                "Runtime": "cloudfront-js-1.0"
            },
            "FunctionMetadata": {
                "FunctionARN": "arn:aws:cloudfront::1234567890:function/url-rewrite-single-page-apps",
                "Stage": "DEVELOPMENT",
                "CreatedTime": "2021-04-09T21:53:20.882000+00:00",
                "LastModifiedTime": "2021-04-09T21:53:21.001000+00:00"
            }
        },
        "ComputeUtilization": "14",
        "FunctionExecutionLogs": [],
        "FunctionErrorMessage": "",
        "FunctionOutput": "{\"request\":{\"headers\":{\"host\":{\"value\":\"www.example.com\"},\"accept\":{\"value\":\"text/html\"}},\"method\":\"GET\",\"querystring\":{\"test\":{\"value\":\"true\"},\"arg\":{\"value\":\"val1\"}},\"uri\":\"/blog/index.html\",\"cookies\":{\"loggedIn\":{\"value\":\"false\"},\"id\":{\"value\":\"CookeIdValue\"}}}}"
    }
}
```

### Using the function

When you’re ready to use your function with a CloudFront distribution, publish the function to the `LIVE` stage. You can do this by updating the `AWS::CloudFront::Function` resource with the `AutoPublish` property set to true. 

```shell
AUTOPUBLISH=true

aws cloudformation deploy \
    --region $REGION \
    --stack-name $STACK \
    --template-file $TEMPLATE \
    --parameter-overrides AutoPublishParam=$AUTOPUBLISH
```

When the function is published to the `LIVE` stage, you can attach it to a distribution’s cache behavior using the function’s ARN in the distributions's CloudFormation template.

CloudFront CloudFormation template references:  
[Distribution](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-distribution.html)  
[DistributionConfig](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cloudfront-distribution-distributionconfig.html)  
[CacheBehavior FunctionAssociations](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cloudfront-distribution-cachebehavior.html#cfn-cloudfront-distribution-cachebehavior-functionassociations)  
[FunctionAssociation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cloudfront-distribution-functionassociation.html)  
