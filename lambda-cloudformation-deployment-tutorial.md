
# Lambda deployment

## 1-Create project files

Create the source go app files

```bash
mkdir src
touch src/main.go
go mod init src
go get github.com/aws/aws-lambda-go/lambda

cat <<EOF > src/main.go
package main

import (
    "context"

    "github.com/aws/aws-lambda-go/lambda"
)

type MyEvent struct {
    Name string `json:"name"`
}

func HandleRequest(ctx context.Context, name MyEvent) (string, error) {
    return "Bye " + name.Name, nil
}

func main() {
    lambda.Start(HandleRequest)
}
EOF

# Compile the application and zip it
GOOS=linux GOARCH=amd64 go build -o main src/main.go
zip deployment.zip main
```

## 2-CloudFormation files

### 2.1-S3 Bucke

Remember to set an unic name for the bucket!
```bash
cat <<EOF > bucket_template.yaml
Parameters:
  BucketName:
    Description: The name of the S3 bucket where the Lambda deployment package will be stored.
    Type: String
    Default: "lambda-template-bucket-sebiuo"

Resources:

  MyS3Bucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Ref BucketName

Outputs:
  S3BucketName:
    Value:
      Ref: MyS3Bucket
    Description: Name of the S3 bucket
EOF

```

### 2.1.1-Apply the bucket template
```bash
# Create the stack
aws cloudformation create-stack --stack-name my-bucket-stack --template-body file://bucket_template.yaml --capabilities CAPABILITY_NAMED_IAM --region us-east-1

# Describe the stack
aws cloudformation describe-stacks --stack-name my-bucket-stack --region us-east-1

# Delete the stack
aws cloudformation delete-stack --stack-name my-bucket-stack --region us-east-1

aws s3 rm s3://lambda-template-bucket-sebiuo --recursive


```
 
## 2.2-Lambda

Now Upload the deployment.zip to the bucket created above

```bash
aws s3 cp deployment.zip s3://lambda-template-bucket-sebiuo/deployment.zip
``` 

Create lambda function template file

```bash
cat <<EOF > lambda_template.yaml
Parameters:
  BucketName:
    Description: The name of the S3 bucket where the Lambda deployment package will be stored.
    Type: String
    Default: "lambda-template-bucket-sebiuo"

  LambdaFunctionName:
    Description: The name of the Lambda function.
    Type: String
    Default: "test-lambda-sebiuo"

  DeploymentFile:
    Description: The name of the Lambda deployment package.
    Type: String
    Default: "deployment.zip"

Resources:

  LambdaExecutionRole:
    Type: "AWS::IAM::Role"
    Properties:
      RoleName: !Sub "${LambdaFunctionName}Role"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal:
              Service:
                - "lambda.amazonaws.com"
            Action:
              - "sts:AssumeRole"
      Path: "/"
      Policies:
        - PolicyName: !Sub "${LambdaFunctionName}Policy"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: "Allow"
                Action:
                  - "logs:CreateLogGroup"
                  - "logs:CreateLogStream"
                  - "logs:PutLogEvents"
                Resource: "arn:aws:logs:*:*:*"
              # Add any other permissions your Lambda might need

  MyLambdaFunction:
    Type: 'AWS::Lambda::Function'
    Properties:
      FunctionName: !Ref LambdaFunctionName
      Handler: main
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        S3Bucket: !Ref BucketName
        S3Key: !Ref DeploymentFile
      Runtime: go1.x
      MemorySize: 128
      Timeout: 10
      Environment:
        Variables:
          lastname: "YourLastName" 

Outputs:
  LambdaFunctionARN:
    Value:
      Fn::GetAtt:
        - "MyLambdaFunction"
        - "Arn"
    Description: ARN of the Lambda function
EOF
```

### 2.2.1-Apply the template

```bash
# For creating the stack:
aws cloudformation create-stack --stack-name my-lambda-stack --template-body file://lambda_template.yaml --capabilities CAPABILITY_NAMED_IAM --region us-east-1

# To describe the stack:
aws cloudformation describe-stacks --stack-name my-lambda-stack --region us-east-1

# For updating the stack:
aws cloudformation update-stack --stack-name my-lambda-stack --template-body file://lambda_template.yaml --capabilities CAPABILITY_NAMED_IAM --region us-east-1

# For deleting the stack:
aws cloudformation delete-stack --stack-name my-lambda-stack --region us-east-1

```

## 3-Test the lambda function

```bash
payload=$(echo '{"Name": "Sebastiano"}' | base64)
aws lambda invoke --function-name test-lambda-sebiuo --payload $payload output.txt --region us-east-1
cat output.txt
```

## 4-To upload a new version of the lambda function

Upload the main.go file to the bucket created

```bash
GOOS=linux GOARCH=amd64 go build -o main src/main.go
zip deployment.zip main
aws s3 cp deployment.zip s3://lambda-template-bucket-sebiuo/deployment.zip
```

Update the lambda function

```bash
aws s3 cp deployment.zip s3://lambda-template-bucket-sebiuo/deployment.zip

aws lambda update-function-code --function-name test-lambda-sebiuo --s3-bucket lambda-template-bucket-sebiuo --s3-key deployment.zip --region us-east-1
```

5 Delete the stack

Delete all objects on the bucket

```bash
aws s3 rm s3://lambda-template-bucket-sebiuo --recursive
```

```bash
aws cloudformation delete-stack --stack-name my-lambda-stack --region us-east-1
aws cloudformation delete-stack --stack-name my-bucket-stack --region us-east-1
``` 
