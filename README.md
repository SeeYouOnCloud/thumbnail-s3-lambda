# thumbnail-s3-lambda
create thumbnails using lambda

Create two s3 buckets 
for eg if: sourcebucket
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