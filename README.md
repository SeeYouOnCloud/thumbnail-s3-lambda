# create thumbnails using lambda


![alt text](https://github.com/SeeYouOnCloud/thumbnail-s3-lambda/blob/main/CreateThumbnails.png?raw=true)

Create two s3 buckets 

For eg: If your source bucket name is "sourcebucket"
then target bucket name should be preceeded by -resized with source bucket name
i.e sourcebucket-resized

Now, Create IAM policy
```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:PutLogEvents",
                "logs:CreateLogGroup",
                "logs:CreateLogStream"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::sourcebucket/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::sourcebucket-resized/*"
        }
    ]
}
```

Then, Create execution IAM role for this policy

Install sam cli 
https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html

Create a folder named lambda-s3

Save the function code as index.js in it
```bash
// dependencies
const AWS = require('aws-sdk');
const util = require('util');
const sharp = require('sharp');
                
// get reference to S3 client
const s3 = new AWS.S3();
                
exports.handler = async (event, context, callback) => {
                
// Read options from the event parameter.
console.log("Reading options from event:\n", util.inspect(event, {depth: 5}));
const srcBucket = event.Records[0].s3.bucket.name;
// Object key may have spaces or unicode non-ASCII characters.
const srcKey    = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, " "));
const dstBucket = srcBucket + "-resized";
const dstKey    = "resized-" + srcKey;
                
// Infer the image type from the file suffix.
const typeMatch = srcKey.match(/\.([^.]*)$/);
if (!typeMatch) {
  console.log("Could not determine the image type.");
  return;
}
                
// Check that the image type is supported
const imageType = typeMatch[1].toLowerCase();
if (imageType != "jpg" && imageType != "png") {
  console.log(`Unsupported image type: ${imageType}`);
  return;
}
                
// Download the image from the S3 source bucket.
                
try {
  const params = {
    Bucket: srcBucket,
    Key: srcKey
  };
  var origimage = await s3.getObject(params).promise();
                
} catch (error) {
  console.log(error);
  return;
}
                
// set thumbnail width. Resize will set the height automatically to maintain aspect ratio.
const width  = 200;
                
// Use the sharp module to resize the image and save in a buffer.
try {
  var buffer = await sharp(origimage.Body).resize(width).toBuffer();
                
} catch (error) {
  console.log(error);
  return;
}
                
// Upload the thumbnail image to the destination bucket
try {
  const destparams = {
    Bucket: dstBucket,
    Key: dstKey,
    Body: buffer,
    ContentType: "image"
  };
                
  const putResult = await s3.putObject(destparams).promise();
                
  } catch (error) {
    console.log(error);
    return;
  }
                
  console.log('Successfully resized ' + srcBucket + '/' + srcKey +
    ' and uploaded to ' + dstBucket + '/' + dstKey);
  };
```

In the lambda-s3 directory, create a node_modules directory

cd node_modules
```bash
npm install --arch=x64 --platform=linux sharp
```

Now, switch to lambda-s3 directory
```bash
zip -r function.zip .
```

Create lambda using this command:
```bash
aws lambda create-function --function-name CreateThumbnail \
--zip-file fileb://function.zip --handler index.handler --runtime nodejs16.x \
--timeout 10 --memory-size 1024 \
--role arn:aws:iam::123456789012:role/lambda-s3-role
```

You can update the timeout using command:
```bash
aws lambda update-function-configuration --function-name CreateThumbnail --timeout 30
```

Test the Lambda function

Create input.txt file in lambda-s3 directory
```bash
{
  "Records":[
    {
      "eventVersion":"2.0",
      "eventSource":"aws:s3",
      "awsRegion":"us-west-2",
      "eventTime":"1970-01-01T00:00:00.000Z",
      "eventName":"ObjectCreated:Put",
      "userIdentity":{
        "principalId":"AIDAJDPLRKLG7UEXAMPLE"
      },
      "requestParameters":{
        "sourceIPAddress":"127.0.0.1"
      },
      "responseElements":{
        "x-amz-request-id":"C3D13FE58DE4C810",
        "x-amz-id-2":"FMyUVURIY8/IgAtTv8xRjskZQpcIZ9KG4V5Wp6S7S/JRWeUWerMUE5JgHvANOjpD"
      },
      "s3":{
        "s3SchemaVersion":"1.0",
        "configurationId":"testConfigRule",
        "bucket":{
          "name":"sourcebucket",
          "ownerIdentity":{
            "principalId":"A3NL1KOZZKExample"
          },
          "arn":"arn:aws:s3:::sourcebucket"
        },
        "object":{
          "key":"HappyFace.jpg",
          "size":1024,
          "eTag":"d41d8cd98f00b204e9800998ecf8427e",
          "versionId":"096fKKXTRTtl3on89fVO.nfljtsv6qko"
        }
      }
    }
  ]
}
```

Invoke the function with the following invoke command
```bash
aws lambda invoke --function-name CreateThumbnail --cli-binary-format raw-in-base64-out --invocation-type Event --payload file://input.txt output.txt
```

Configure Amazon S3 to publish events
```bash
aws lambda add-permission --function-name CreateThumbnail --principal s3.amazonaws.com \
--statement-id s3invoke --action "lambda:InvokeFunction" \
--source-arn arn:aws:s3:::sourcebucket \
--source-account account-id
```

Now in your s3 source bucket set trigger

Under properties tab -> Event notifications, choose Create event notification to configure a notification with the following settings:
```bash
Event name – lambda-trigger

Event types – All object create events

Destination – Lambda function

Lambda function – CreateThumbnail
```

Now test if trigger works

Upload a file in source bucket and check in target bucket if thumbnail is created
