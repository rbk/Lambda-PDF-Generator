# Building a PDF Generator on AWS Lambda with Python3 and wkhtmltopdf

## Introduction

Creating scalable APIs in 2019 is easier than ever before with serverless auto-scaling compute power being widely accessible. In this article I'll show you how I created a PDF generator API that can handle with 1000 concurrent requests.

The purpose of this artice is to show you how I accomplished creating the API so that you will see how easy it is to create your own serverless APIs. When this project is complete, you will have an API endpoint in which you can POST a JSON object to, and recieve a link to a PDF on Amazon S3. The JSON object looks like this:

```json
{
  "filename": "sample.pdf",
  "html": "<html><head></head><body><h1>It works! This is the default PDF.</h1></body></html>"
}
```

## Setup

**What you'll need:**

- Python3
- Aws CLI installed and configured
- Serverless installed

### Serverless

The first thing you need to do is initialize a new Serverless project. Serverless is a tool that greatly simplifies using AWS, Gcloud, and Azure services so you can create APIs without worrying about managing a server. If you need more information about Serverless visite their website: [Serverless.com](https://serverless.com). In your terminal run the following command to initialize your project.

**Note: sls is short for serverless and can be used interchangably.**

```zsh
sls create --template aws-python3
```

This command will bootstrap a Python3 Lambda setup for you to work from.

### WKHTMLTOPDF Binary

Next we need to get the binary for turning HTML into PDFs. For this I used [WKHTMLTOPDF](https://github.com/wkhtmltopdf/wkhtmltopdf). **Important:** You have to use version 0.12.4 becaused later versions of wkhtmltopdf require dynamic dependencies of the host system that cannot be installed on Lambda.

Download version 0.12.4 here:

https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.4/wkhtmltox-0.12.4_linux-generic-amd64.tar.xz

Once you have extract this tar file, copy the binary ***wkhtmltopdf*** to the binary folder in your project.

```
./binary/wkhtmltopdf
```

**More information about wkhtmltopdf can be found on their website:**
[WKHTMLTOPDF](https://wkhtmltopdf.org/)

### Python3 Dependencies

Now that we have the WKHTMLtoPDF binary, we need the Python library, ***pdfkit*** to use it. Since we are using Serverless and AWS Lambda we cannot just run *pip install pdfkit*. We need a Serverless plugin to install our dependencies on Lambda.

In our project folder install the **python plugin requirements** module for Serverless.

```zsh
sls plugin install -n serverless-python-requirements
```
Now, in your serverless.yml file, you need to add a custom section in the yaml:

```yaml
custom:
  pythonRequirements:
    dockerizePip: true
```

Once the serverless plugin requirements is installed, you can add a requirements.txt file to your project and it will be automatically installed on lambda when you deploy.

Your requirements.txt for this project only needs to have pdfkit.

**Requirements.txt**
```text
pdfkit
```

**For any issues with this module checkout the repository issue:**
[Serverless Python Requirements](https://github.com/UnitedIncome/serverless-python-requirements)

## Ready, Set, Code

We are ready to code now. We only need to work with two files for this project: serverless.yml ad handler.py. Typically you want your serverless projects to be lightweight and independent of other parts of your codebase.

### Serverless.yml

The **serverless.yml** file is our main configuration file. A serverless.yml file is broken down into a few important sections. If you are new to YAML, [checkout the documentation for YAML](https://learn.getgrav.org/16/advanced/yaml).

For this serverless.yml configuration you'll need to create a file called config.yml. This will store the S3 bucket name. The serverless.yml will reference the config.yml to setup the correct bucket for your project.

**Contents of config.yml**
```
BucketName: 'your-s3-bucket-name'
```

**Here is a high level overview of a serverless.yml file:**

```
service: pdf-services # name or reference to our project
provider: # It is possible to use Azure, GCloud, or AWS
functions: # Array of functions to deploy as Lambdas
resources: # S3 buckets, DynamoDB tables, and other possible resources to create
plugins: # Plugins for Serverless
custom: # Custom variables used by you or plugins during setup and deployment

```

Our serverless configuration will do a few things for us when we deploy:

1. Create an S3 bucket called **pdf-service-bucket** to store our PDFs
2. Create a function that will create the PDFs
3. Give our function access to the S3 bucket
4. Setup an API endpoint for our Lambda function at: 
```
POST https://xxxx.execute-api.xxxx.amazonaws.com/dev/new-pdf
```

Here is the full serverless.yml configuration. I've added a couple important comments in the code.

```yaml
service: pdf-service
provider:
  name: aws
  runtime: python3.7
  # Set environment variable for the S3 bucket
  environment:
    S3_BUCKET_NAME: ${file(./config.yml):BucketName}
  # Gives our functions full read and write access to the S3 Bucket
  iamRoleStatements:
    -  Effect: "Allow"
       Action:
         - "s3:*"
       Resource:
          - arn:aws:s3:::${file(./config.yml):BucketName}
          - arn:aws:s3:::${file(./config.yml):BucketName}/*
functions:
  generate_pdf:
    handler: handler.generate_pdf
    events:
      - http:
          path: new-pdf
          method: post
          cors: true
resources:
 # Creates an S3 bucket in our AWS account
 Resources:
   NewResource:
     Type: AWS::S3::Bucket
     Properties:
       BucketName: ${file(./config.yml):BucketName}
custom:
  pythonRequirements:
    dockerizePip: true
plugins:
  - serverless-python-requirements

```

### Handler.py

Handler.py is the only Python file we need. It contains one function to generate the PDF and save it to Lambda. You can creat more functions in this file to split the code into reusable parts, but for this example one function was enough. Lambda functions take two arguments by default: Event and Context.

**Context** contains environment variables and system information.

**Event** contains request data that is sent to the lambda function.

In this project we will send our function **generate_pdf** a filename and HTML, and it will return the URI for a PDF it creates.

```python
import json
import pdfkit
import boto3
import os
client = boto3.client('s3')

# Get the bucket name environment variables to use in our code
S3_BUCKET_NAME = os.environ.get('S3_BUCKET_NAME')

def generate_pdf(event, context):
    
    # Defaults
    key = 'deafult-filename.pdf'
    html = "<html><head></head><body><h1>It works! This is the default PDF.</h1></body></html>"
    
    # Decode json and set values for our pdf    
    if 'body' in event:
        data = json.loads(event['body'])
        key = data['filename']
        html = data['html'] 

    # Set file path to save pdf on lambda first (temporary storage)
    filepath = '/tmp/{key}'.format(key=key)
    
    # Create PDF
    config = pdfkit.configuration(wkhtmltopdf="binary/wkhtmltopdf")
    pdfkit.from_string(html, filepath, configuration=config, options={})
    

    # Upload to S3 Bucket
    r = client.put_object(
        ACL='public-read',
        Body=open(filepath, 'rb'),
        ContentType='application/pdf',
        Bucket=S3_BUCKET_NAME,
        Key=key
    )
    
    # Format the PDF URI
    object_url = "https://{0}.s3.amazonaws.com/{1}".format(S3_BUCKET_NAME, key)

    # Response with result
    response = {
        "headers": {
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Credentials": True,
        },
        "statusCode": 200,
        "body": object_url
    }

    return response
```

## Deploy

Now that you have all the code ready it is time to deploy. In your project folder run the following command from your terminal:

```zsh
sls deploy
```
After you run deploy, Serverless will create everything for you in AWS. You will get HTTP POST endpoint that you will use to generate PDFs. The endpoint will look something like this:
```zsh
https://xxxxxx.execute-api.us-east-1.amazonaws.com/dev/new-pdf
``` 

You can use **curl** to test your function. The following curl command posts a JSON object to the lambda endpoint. The JSON object contains a filename and some html to turn into a PDF.

```zsh
curl -d '{"filename":"my-sample-filename.pdf", "html":"<html><head></head><body><h1>Custom HTML -> Posted From CURL as {JSON}</h1></body></html>"}' -H "Content-Type: application/json" -X POST REPLACE-WITH-YOUR-ENDPOINT
```

*Note: Replace "REPLACE-WITH-YOUR-ENDPOINT" with the endpoint you receive from Serverless.*

After running this command you should receive the URI to your generated PDF.


## Conclusion

Creating a scalable PDF generator is easy with of AWS and Serverless. In addition to having a scalable API, you don't have to worry about server maintenance. I encourage you to create more projects with AWS Lambda to get comfortable with the platform and configuration.

Thanks for reading!


## Next Steps

- Setup a custom domain with [Serverless Domain Manager](https://github.com/amplify-education/serverless-domain-manager)
- Setup local testing with [Serverless Offline](https://github.com/dherault/serverless-offline)

## Resources

- Serverless Environment Variables - https://serverless.com/framework/docs/providers/aws/guide/variables/
- Serverless CORS Guide - https://serverless.com/blog/cors-api-gateway-survival-guide/
- Fonts AWS Lambda - https://forums.aws.amazon.com/thread.jspa?messageID=776307
- Boto3 API - https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#S3.Client.upload_file
- S3 Argument Reference - https://gist.github.com/rbk/926bfd3d886b2c25c53818eeb6e77d6a