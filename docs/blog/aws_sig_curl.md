<title>Signing AWS Sig V4 requests using curl and Postman</title>

# Signing AWS Sig V4 requests using curl and Postman
![AWS](  https://img.shields.io/badge/Amazon_AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
&nbsp;

Curl and Postman both supports AWS request signing - using AWS Sig V4 method - with IAM credentials. Using these tools is often more convenient than setting up scripts using AWS SDKs.

This text covers how to make signed requests to an AWS endpoint with IAM user and role credentials, both assuming the role locally, and via an EC2 instance profile.

While the curl [documentation](https://curl.se/libcurl/c/CURLOPT_AWS_SIGV4.html) shows how to sign requests as an IAM user; IAM role credentials can also be used, by adding an extra header (i.e. "*x-amz-security-token*") to the request, containing the session token.

&nbsp;

#### Format of a signed curl request as IAM user:

```
curl --request GET "https://<SERVICE_DOMAIN_ENDPOINT>/" --user "<AWS_IAMUSER_ACCESS_KEY>:<AWS_IAMUSER_SECRETE_KEY>" --aws-sigv4 "aws:amz:<REGION>:es"

```
&nbsp;

#### Format of a signed curl request as IAM role (or any type of short-lived credentials):

```
curl --request GET "https://<SERVICE_ENDPOINT>/" --user "<AWS_IAMUSER_ACCESS_KEY>:<AWS_IAMUSER_SECRETE_KEY>" --aws-sigv4 "aws:amz:<REGION>:es" -H "x-amz-security-token:<SESSION_TOKEN>"
```
&nbsp;

#### Example 1. Signing requests using an assumed IAM role credentials

Let's assume we have a service with a resource policy in a domain like the following (i), which forces us to sign requests with IAM role "curlRole" credentials.

(i)

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "resource permissions",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<AWS_ACCOUNT>:role/curlRole"
            },
            "Action": "es:ESHttp*",
            "Resource": [
                "arn:aws:es:<REGION>:<AWS_ACCOUNT>:<RESOURCE>/<RESOURCE_NAME>/*"
            ]
        }
    ]
}

```
&nbsp;

To make a signed curl request as an IAM role, we need three parameters: the IAM role's **AccessKeyId**, **SecretAccessKey** and **SessionToken**. One way to get these parameters is to assume the IAM role in question. For example, [**sts assume role** AWS CLI command](https://awscli.amazonaws.com/v2/documentation/api/2.0.33/reference/sts/assume-role.html) can be used to retrieve this information.

The following command stores the **AccessKeyId**, **SecretAccessKey** and **SessionToken** of the IAM role in question (in this case, "curlRole"), from the output of **sts assume role** command, to be used later in the curl request.

Note: In order to assume the role, the caller IAM entity must have **sts:AssumeRole** permission on the IAM role "curlRole".

```
read -r AccessKeyId SecretAccessKey SessionToken <<< $(aws sts assume-role --role-arn <curlRole_ARN> --role-session-name curlTest | jq -r --argjson fields '["AccessKeyId","SecretAccessKey","SessionToken"]' '[.Credentials[$fields[]]] | join("\n")')

```
&nbsp;

Then, we can make the signed curl request to the AWS endpoint, as IAM "curlRole", and confirm that the request was successfully authenticated.

```
curl --request GET "https://<SERVICE_DOMAIN_ENDPOINT>/" --user "${AccessKeyId}:${SecretAccessKey}" --aws-sigv4 "aws:amz:<REGION>:es" -H "x-amz-security-token:${SessionToken}"

```
&nbsp;

#### Example 2. Signing requests from an EC2 instance, using the instance profile credentials

To sign curl requests from an EC2 instance, using the instance profile's role, we should first retreive the instance profile's credentials and session token from the [instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html).

First, we must retrieve the instance profile role Id from the metadata.

```
TOKEN=`curl -sS -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
ROLE=`curl -sS -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/`

```
&nbsp;


Next, we retrieve the instance profile role **AccessKeyId**, **SecretAccessKey** and **SessionToken**.

```
ACCESS_KEY_ID=`curl -sS -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE | jq -r '.AccessKeyId'`
SECRET_ACCESS_KEY=`curl -sS -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE | jq -r '.SecretAccessKey'`
SESSION_TOKEN=`curl -sS -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE | jq -r '.Token'`

```
&nbsp;

With  this information, we can build the request as [shown previously](#format-of-a-signed-curl-request-as-iam-role-or-any-type-of-short-lived-credentials).

&nbsp;
#### Signing requests using Postman

Knowing the process to sign requests both with long-lived (e.g. IAM users) and short-lived (e.g. IAM roles) credentials, we can apply the same principle to Postman requests. For step-by-step instructions, you can check out their [documentation](https://learning.postman.com/docs/sending-requests/authorization/aws-signature/) on the topic.

You can find more information on AWS SigV4, and it's integration with cURL and Postman below.

#### Resources
- [AWS Request Signing](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-signing.html)
- [Curl AWS Signature](https://curl.se/libcurl/c/CURLOPT_AWS_SIGV4.html)
- [Postman AWS Signature](https://learning.postman.com/docs/sending-requests/authorization/aws-signature/)