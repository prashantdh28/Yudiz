provider "aws" {
  region     = "ap-south-1"
  access_key = "AKIAWL3SPHGY5PCPQNS4"
  secret_key = "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"
}

# create vpc
resource "aws_vpc" "cloud" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "cloudvpc"
  }
}

# create subnets
resource "aws_subnet" "public-subnet" {
  vpc_id     = aws_vpc.cloud.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_subnet" "private-subnet" {
  vpc_id     = aws_vpc.cloud.id
  cidr_block = "10.0.2.0/24"

  tags = {
    Name = "private-subnet"
  }
}

# create security group
resource "aws_security_group" "cloudsg" {
  name        = "cloudsg"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_vpc.cloud.id

  ingress {
    description = "TLS from VPC"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "cloud-sg"
  }
}

# create internet gateway
resource "aws_internet_gateway" "cloud-igw" {
  vpc_id = aws_vpc.cloud.id

  tags = {
    Name = "cloud-igw"
  }
}

# create route table
resource "aws_route_table" "public-rt" {
  vpc_id = aws_vpc.cloud.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.cloud-igw.id
  }


  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table" "private-rt" {
  vpc_id = aws_vpc.cloud.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.cloud-nat.id
  }


  tags = {
    Name = "private-rt"
  }
}

# create route table association
resource "aws_route_table_association" "private-asso" {
  subnet_id      = aws_subnet.private-subnet.id
  route_table_id = aws_route_table.private-rt.id
}

resource "aws_route_table_association" "public-asso" {
  subnet_id      = aws_subnet.public-subnet.id
  route_table_id = aws_route_table.public-rt.id
}

# create key-pair
resource "aws_key_pair" "yudizkey" {
  key_name   = "yudizkey"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC8SAK+dcmSdu0y9/V0MSW+XWttLf2ok5nge8Zhg+zu0CtgKL8XNjxHjmc0qixlix7QGKhw7gxNIE4Sti6ybK13H/2CRU8ZnITAZ185t0PC3xMtnyNgxLpW72KB8Z1unSUcIPvLP50NdRG0Ksm/b0bXDqv5JEctvqYda7JaZT7/AVBJH9u8Ftbxa5T1FvbWg2nfxVG9u1S8dRc8pukVNLtyvHj+G/D8Fa+ld/B1Q8m2cMnwA1Xs0F2u1rOeUPcl/hIGArlP49K0FmOBxhFA9MfAzwjGoPmIQSah2dvxQWPg3Wl9BWNw1iQTY2bDhksGD36rlnWDHE0CqOZC7XgI6fRj root@ip-172-31-12-31.ap-south-1.compute.internal"
}
#create instance
resource "aws_instance" "instance" {
  ami                    = "ami-01a4f99c4ac11b03c"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public-subnet.id
  vpc_security_group_ids = [aws_security_group.cloudsg.id]
  key_name               = "yudizkey"

  tags = {
    Name = "instance"
  }
}

resource "aws_instance" "db-instance" {
  ami                    = "ami-01a4f99c4ac11b03c"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.private-subnet.id
  vpc_security_group_ids = [aws_security_group.cloudsg.id]
  key_name               = "yudizkey"

  tags = {
    Name = "db-instance"
  }
}

resource "aws_eip" "cloud-natip" {
  vpc = true
}

resource "aws_nat_gateway" "cloud-nat" {
  allocation_id = aws_eip.cloud-natip.id
  subnet_id     = aws_subnet.public-subnet.id
}

resource "aws_security_group" "elb" {
  name = "teraform-example-elb"
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Create a Launch Configuration
resource "aws_launch_configuration" "ubuntu" {
  image_id        = "ami-0f8ca728008ff5af4"
  instance_type   = "t2.micro"
  key_name        = "yudizkey"
  security_groups = ["sg-03726efbae8345350"]
}

# Create an Auto Scaling Group
resource "aws_autoscaling_group" "ubuntu" {
  name                 = "ubuntu"
  launch_configuration = aws_launch_configuration.ubuntu.id
  vpc_zone_identifier  = ["subnet-00ba1f77c313f378a","subnet-0eda9fa948d0e80fa"]
  min_size             = 1
  max_size             = 3
  desired_capacity     = 2
}
# Create a scaling policy
resource "aws_autoscaling_policy" "my-cpu-scaling-policy" {
  name                   = "my-cpu-scaling-policy"
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = 1
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.ubuntu.name
}

#create MySQL RDS
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.27"
    }
  }

  required_version = ">= 0.14.9"
}

resource "aws_db_instance" "rds_instance" {
  allocated_storage   = 20
  identifier          = "rds-terraform"
  storage_type        = "gp2"
  engine              = "mysql"
  engine_version      = "8.0.27"
  instance_class      = "db.t2.micro"
  name                = "mysqldb"
  username            = "admin"
  password            = "admin123"
  publicly_accessible = true
  skip_final_snapshot = true


  tags = {
    Name = "ExampleRDSServerInstance"
  }
}

# create a target group
resource "aws_lb_target_group" "target_group" {
  health_check {
    interval            = 10
    path                = "/"
    protocol            = "HTTP"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }

  name        = "whiz-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "instance"
  vpc_id      = aws_vpc.cloud.id
}


#create cloud watch
resource "aws_cloudwatch_dashboard" "EC2-Dashboard" {
  dashboard_name = "EC2-Dashboard"
  dashboard_body = <<EOF
{
    "widgets": [
        {
            "type": "explorer",
            "width": 24,
            "height": 15,
            "x": 0,
            "y": 0,
            "properties": {
                "metrics": [
                    {
                        "metricName": "CPUUtilization",
                        "resourceType": "AWS::EC2::Instance",
                        "stat": "Maximum"
                    }
                ],
                "aggregateBy": {
                    "key": "InstanceType",
                    "func": "MAX"
                },
                "labels": [
                    {
                        "key": "State",
                        "value": "running"
                    }
                ],
                "widgetOptions": {
                    "legend": {
                        "position": "bottom"
                    },
                    "view": "timeSeries",
                    "rowsPerPage": 8,
                    "widgetsPerRow": 2
                },
                "period": 60,
                "title": "Running EC2 Instances CPUUtilization"
            }
        }
    ]
}
EOF
}
# create cloudwatch alarm
resource "aws_cloudwatch_composite_alarm" "EC2" {
  alarm_description = "Composite alarm that monitors CPUUtilization "
  alarm_name        = "EC2_Composite_Alarm"
  alarm_actions     = [aws_sns_topic.EC2_topic.arn]

  alarm_rule = "ALARM(${aws_cloudwatch_metric_alarm.EC2_CPU_Usage_Alarm.alarm_name})"

  depends_on = [
    aws_cloudwatch_metric_alarm.EC2_CPU_Usage_Alarm,
    aws_sns_topic.EC2_topic,
    aws_sns_topic_subscription.EC2_Subscription
  ]
}


# Creating the AWS CLoudwatch Alarm that will autoscale the AWS EC2 instance based on CPU utilization.
resource "aws_cloudwatch_metric_alarm" "EC2_CPU_Usage_Alarm" {
  alarm_name          = "EC2_CPU_Usage_Alarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "2"
  metric_name = "CPUUtilization"
  namespace = "AWS/EC2"
  period    = "60"
  statistic = "Average"
  threshold         = "70"
  alarm_description = "This metric monitors ec2 cpu utilization exceeding 70%"

}
# create a cloud watch log group
resource "aws_cloudwatch_log_group" "ebs_log_group" {
  name              = "ebs_log_group"
  retention_in_days = 30
}


resource "aws_cloudwatch_log_stream" "ebs_log_stream" {
  name           = "ebs_log_stream"
  log_group_name = aws_cloudwatch_log_group.ebs_log_group.name
}

# create sns topic and subscription
resource "aws_sns_topic" "EC2_topic" {
  name = "EC2_topic"
}

resource "aws_sns_topic_subscription" "EC2_Subscription" {
  topic_arn = aws_sns_topic.EC2_topic.arn
  protocol  = "email"
  endpoint  = "prashantdhole7620@gmail.com"

  depends_on = [
    aws_sns_topic.EC2_topic
  ]
}
