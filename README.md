# Lambda access to S3 via VPC Endpoint

AWS CloudFormation script that demonstrates a Lambda function running within a VPC and accessing S3 using a VPC Endpoint.

The script creates an S3 bucket, and a Lambda function that creates a record within that bucket.  The Lambda is associated
to a VPC that only contains private subnets (i.e. there are no Internet/NAT Gateways) and a VPC Endpoint to S3, allowing
access to the S3 bucket only.

The VPC that the Lambda function is associated with is created using the script in [VPC](https://github.com/gford1000-aws/vpc),
creating up to 6 private subnets (to which the Lambda is associated) with a CIDR of your choice.

The script creates a nested stack, constructing the VPC separately from the Lambda and S3 bucket, for clarity.  

Notes:

1. the VPC *must* have ```EnableDnsSupport = true``` so that DNS resolution of URLs can be performed.

2. the Lambda IAM Role includes ```ec2:CreateNetworkInterface```, ```ec2:DescribeNetworkInterfaces```, ```ec2:DeleteNetworkInterface``` to allow the ENI to be created within the VPC, as well as the necessary S3 permission (s3:PutObject) to create the record in the bucket.

3. the Lambda Security Group only allows egress via the VPC EndPoint.

4. the policy of the VPC EndPoint only allows access create records in the S3 bucket, but allows this for all principals.  This can be restricted to Lambda only if required.

5. unfortunately, CloudFormation does not return the prefix list value for the VPC Endpoint service, so this must be passed to the script.  The value can be
found using the AWS CLI: [aws ec2 describe-prefix-lists](http://docs.aws.amazon.com/cli/latest/reference/ec2/describe-prefix-lists.html)


## Boto3 Specific Notes:

By default, Boto3 uses *virtual* S3 urls.  As a result, they require resolution to the region specific url, and the resolution step requires internet access.  This
causes hanging of the Lambda function (until it times out) since there is no internet access.

To ensure the Lambda can reach S3, Boto3 must use a *path* ```addressing_style``` via the ```Config``` object:

```python
import boto3
import botocore.config

client = boto3.client('s3', 'ap-southeast-2', config=botocore.config.Config(s3={'addressing_style':'path'}))
```

Notes:

1. The specified AWS region must correspond to the region in which the Lambda and VPC Endpoint have been deployed.

2. The use of the ```Config``` object is benign: if you remove the VPC association, then the Lambda will continue to work via the the internet.

3. For more details on S3 urls, read the AWS documentation on [virtual S3 urls](http://docs.aws.amazon.com/AmazonS3/latest/dev/VirtualHosting.html).

4. For more details on boto3 configurations, see [botocore.config](http://botocore.readthedocs.io/en/latest/reference/config.html#botocore-config).


## Arguments

| Argument             | Description                                                        |
| -------------------- |:------------------------------------------------------------------:|
| CidrAddress          | First 2 elements of CIDR block, which is extended to be X.Y.0.0/16 |
| PrivateSubnetCount   | The number of private subnets to be created (2-6 can be selected)  |
| S3EndpointPrefixList | The pl-xxxxxxx identifier for the S3 end point in the region       |
| VPCTemplateURL       | The S3 url to the VPC Cloudformation script                        |


## Outputs

| Output                  | Description                                                 |
| ----------------------- |:-----------------------------------------------------------:|
| Bucket                  | The name of the S3 bucket to which the Lambda will write    |
| Lambda                  | The name of the Lambda function                             |
| VPC                     | The reference to the VPC                                    |


## Licence

This project is released under the MIT license. See [LICENSE](LICENSE) for details.
