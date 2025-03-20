provider "aws" {
  region = "us-east-1" # Change region if necessary
}

# S3 Bucket to store IoT data
resource "aws_s3_bucket" "iot_energy_monitoring" {
  bucket = "iot-energy-monitoring-datas"
  # object_ownership = "BucketOwnerEnforced" 
}

# Server-Side Encryption for S3
resource "aws_s3_bucket_server_side_encryption_configuration" "iot_data_sse" {
  bucket = aws_s3_bucket.iot_energy_monitoring.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Enable versioning for S3 bucket
resource "aws_s3_bucket_versioning" "iot_data_versioning" {
  bucket = aws_s3_bucket.iot_energy_monitoring.id
  versioning_configuration {
    status = "Enabled"
  }
}

# DynamoDB Table for real-time monitoring
resource "aws_dynamodb_table" "iot_metrics" {
  name           = "IoTMetrics"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "device_id"
  range_key      = "timestamp"

  attribute {
    name = "device_id"
    type = "S"
  }

  attribute {
    name = "timestamp"
    type = "N"
  }

  ttl {
    attribute_name = "ttl"
    enabled        = true
  }
}

# IAM Role for IoT permissions
resource "aws_iam_role" "iot_role" {
  name = "IoTDataRole"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "iot.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}

resource "aws_iam_role" "lambda_exec_role" {
  name = "lambda_exec_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

# IAM Policy for IoT Role
resource "aws_iam_policy" "iot_policy" {
  name = "IoTDataPolicy"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "dynamodb:PutItem"
      ],
      "Resource": [
        "${aws_s3_bucket.iot_energy_monitoring.arn}",
        "${aws_dynamodb_table.iot_metrics.arn}"
      ]
    }
  ]
}
EOF
}

# IAM Policy for Lambda Execution
resource "aws_iam_policy" "lambda_policy" {
  name = "LambdaExecutionPolicy"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "${aws_s3_bucket.iot_energy_monitoring.arn}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "${aws_dynamodb_table.iot_metrics.arn}"
    }
  ]
}
EOF
}


# Attach Policy to Role
resource "aws_iam_role_policy_attachment" "iot_role_attach" {
  role       = aws_iam_role.iot_role.name
  policy_arn = aws_iam_policy.iot_policy.arn
}

resource "aws_iam_role_policy_attachment" "lambda_role_attach" {
  role       = aws_iam_role.iot_role.name
  policy_arn = aws_iam_policy.lambda_policy.arn
}

# AWS IoT Topic Rule to send messages to S3 and DynamoDB
resource "aws_iot_topic_rule" "iot_to_s3" {
  name        = "iot_to_s3_rule"
  sql         = "SELECT * FROM 'iot/energy'"
  sql_version = "2016-03-23"  
  enabled     = true

  s3 {
    role_arn    = aws_iam_role.iot_role.arn
    bucket_name = aws_s3_bucket.iot_energy_monitoring.bucket
    key         = "iot-data/${timestamp()}.json"
  }

  dynamodb {
    role_arn       = aws_iam_role.iot_role.arn
    table_name     = aws_dynamodb_table.iot_metrics.name
    hash_key_field = "device_id"
    hash_key_value = "$${clientId()}" # Fix here
    range_key_field = "timestamp"
    range_key_value = "$${timestamp()}"
  }
}

# Lambda Function for additional processing
resource "aws_lambda_function" "process_iot_data" {
  function_name = "process_iot_data"
  runtime       = "python3.9"
  role          = aws_iam_role.lambda_exec_role.arn
  handler       = "lambda_function.lambda_handler"
  filename      = "lambda.zip"

  source_code_hash = filebase64sha256("lambda.zip")
}

# IoT Rule to trigger Lambda
resource "aws_iot_topic_rule" "iot_to_lambda" {
  name        = "iot_to_lambda_rule"
  sql         = "SELECT * FROM 'iot/energy'"
  sql_version = "2016-03-23"  
  enabled     = true

  lambda {
    function_arn = aws_lambda_function.process_iot_data.arn
  }
}

# Lambda Permission to allow invocation from IoT
resource "aws_lambda_permission" "allow_iot" {
  statement_id  = "AllowExecutionFromIoT"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.process_iot_data.function_name
  principal     = "iot.amazonaws.com"
}

# Outputs
output "s3_bucket" {
  value = aws_s3_bucket.iot_energy_monitoring.bucket
}

output "dynamodb_table" {
  value = aws_dynamodb_table.iot_metrics.name
}
