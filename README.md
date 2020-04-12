# ThingsBoard-AWS-EB

[ThingsBoard](https://thingsboard.io/) dockerized application deployment to [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/).

![ThingsBoard](thingsboard.gif)
![AWS Elastic Beanstalk + Docker](aws_eb-docker.jpeg)

# ThingsBoard

[ThingsBoard](https://thingsboard.io/docs/getting-started-guides/what-is-thingsboard/) is an open-source IoT platform that enables rapid development, management and scaling of IoT projects. Our goal is to provide the out-of-the-box IoT cloud or on-premises solution that will enable server-side infrastructure for your IoT applications.

## Local Execution

To run locally the dockerized ThingsBoard application, follow the instructions described in these links:

* [Installing ThingsBoard CE using Docker (Linux or Mac OS)](https://thingsboard.io/docs/user-guide/install/docker/)
* [thingsboard/tb-postgres - Docker Hub](https://hub.docker.com/r/thingsboard/tb-postgres/)

# Deployment

## How does it work?

The `deploy.py` CLI deploys the ThingsBoard application to AWS Elastic Beanstalk performing the following tasks:

1. Create a boto3 session to store the AWS credentials and the region to use for the deployment. If there is any configuration defined through [environment variables](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html) or stored in AWS CLI `~/.aws` folder, the AWS credentials and the region are directly retrieved from there (it is possible to specify the profile to use as optional argument). Otherwise, the CLI asks for these parameters interactively.
2. Check if the Elastic Beanstalk application is already created, and if not, create it.
3. Create the ZIP package (source bundle) for the new application version.
4. Upload the application version package to a source bundle S3 bucket.
5. Create the new application version for the Elastic Beanstalk application.
6. Create the environment if it does not exist or has been previously terminated, or update it if it is already created. The CLI check if there is an operation in progress in the environment and wait for it to be completed before updating or recreating it.

## Dependencies

* `Python 3.3+`
* `boto3`

## Prerequisites

* The `aws-elasticbeanstalk-ec2-role` instance profile (AWS IAM EC2 Role) created in the AWS account:

    [Managing Elastic Beanstalk instance profiles](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/iam-instanceprofile.html)

* The `aws-elasticbeanstalk-service-role` service role (AWS IAM Elastic Beanstalk Role) created in the AWS account:

    [Managing Elastic Beanstalk service roles](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/iam-servicerole.html)

* An IAM User or an AWS IAM Role attached to the EC2 instance (only for executions from EC2 instances) with the following IAM Managed Policy attached:

      AWSElasticBeanstalkFullAccess

* Pip tool for Python packages management. Installation:

      # From Linux
      $ curl -O https://bootstrap.pypa.io/get-pip.py
      $ sudo python3 get-pip.py

      # From macOS
      $ brew install python

* AWS SDK for Python. Installation:

      # From Linux
      $ sudo pip3 install boto3

      # From macOS
      $ pip3 install boto3

## Execution Method

Here you have the message that you will get if you request help to the `deploy.py` CLI:

    $ ./deploy.py --help
    usage: deploy.py [-h] -a APPLICATION_NAME -e ENVIRONMENT_NAME [-p PROFILE]

    Custom CLI to deploy an application to AWS Elastic Beanstalk

    optional arguments:
      -h, --help            show this help message and exit

    Options:
      -a APPLICATION_NAME, --application-name APPLICATION_NAME
                            Name of the Elastic Beanstalk application
      -e ENVIRONMENT_NAME, --environment-name ENVIRONMENT_NAME
                            Name of the environment for the Elastic Beanstalk
                            application
      -p PROFILE, --profile PROFILE
                            Use a specific profile from AWS CLI stored
                            configurations

## Provided Configuration Profiles

This project provides the following configuration profiles:

* `env.yaml.staging` => Configuration profile intended for staging environments deployed for development, test, QA or demo purposes (lower cost and performance). This profile allows to deploy a single-instance type environment (henceforth `SingleInstance`) made up of one EC2 instance with an Elastic IP address to serve the application, and a Single-AZ RDS instance to host the databases.
* `env.yaml.production` => Configuration profile intended for environments with productive loads. This profile allows to deploy a load-balancing, autoscaling type environment (henceforth `LoadBalanced`) made up of an Auto Scaling group of EC2 instances with an ELB to serve the application, and a Multi-AZ RDS instance to host the databases.

## Execution Examples

    # Deployment of a staging environment
    $ ln -sf env.yaml.staging env.yaml
    $ ./deploy.py --application-name thingsboard --environment-name demo [--profile <aws_cli_profile>]

    # Deployment of a productive environment
    $ ln -sf env.yaml.production env.yaml
    $ ./deploy.py --application-name thingsboard --environment-name live [--profile <aws_cli_profile>]

## Environment Customization

If you need to modify any settings provided in this project or add new ones, feel free to fork or import this repository and apply the changes you need to fulfil your use case.

Here are some common customizations that you might want to add to an environment:

* To deploy a `SingleInstance` type environment in a existing custom VPC, add the following configuration block to the `env.yaml.staging` file:

        aws:ec2:vpc:
          VPCId: <vpc_id>
          Subnets: <subnet_1a_id>,<subnet_1b_id>,...      # Subnet IDs for EC2 instance
          DBSubnets: <subnet_2a_id>,<subnet_2b_id>,...    # Subnet IDs for RDS instance

* To deploy a `LoadBalanced` type environment in a existing custom VPC, add the following settings to the existing `aws:ec2:vpc` namespace in `env.yaml.production` file:

        aws:ec2:vpc:
          ELBScheme: public                               # This setting is already defined
          VPCId: <vpc_id>
          ELBSubnets: <subnet_3a_id>,<subnet_3b_id>,...   # Subnet IDs for ELB
          Subnets: <subnet_1a_id>,<subnet_1b_id>,...      # Subnet IDs for Auto Scaling group
          DBSubnets: <subnet_2a_id>,<subnet_2b_id>,...    # Subnet IDs for RDS instance
          AssociatePublicIpAddress: true                  # Review this option in https://amzn.to/2Vhgt5B

* To configure HTTPS in a `LoadBalanced` type environment for a custom domain using an ACM issued certificate, add the following configuration block to the `env.yaml.production` file:

        aws:elb:listener:443:
          ListenerProtocol: HTTPS
          InstancePort: '80'
          InstanceProtocol: HTTP
          ListenerEnabled: true
          SSLCertificateId: <acm_certificate_arn>

    Then uncomment the `https-redirect-docker-sc` container command block in `.ebextensions/server-updates.config` file:

        https-redirect-docker-sc:
          command: cp .ebextensions/nginx/elasticbeanstalk-nginx-docker-proxy.conf /etc/nginx/sites-available/
          test: '[ "$(/opt/elasticbeanstalk/bin/get-config environment -k CONFIG_PROFILE)" == "production" ]'

* To configure a SSH key pair to securely log into the EC2 instance/s belonging to an environment, add the following setting to the existing `aws:autoscaling:launchconfiguration` namespace in any of the `env.yaml.<config_profile>` files:

        aws:autoscaling:launchconfiguration:
          . . .
          . . .
          EC2KeyName: <key_pair_name>

## Related Links

* [Installing ThingsBoard CE on CentOS/RHEL](https://thingsboard.io/docs/user-guide/install/rhel/)
* [Single Container Docker Configuration](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/single-container-docker-configuration.html)
* [Environment Manifest (env.yaml)](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-cfg-manifest.html)
* [Environment types](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features-managing-env-types.html)
* [General options for all environments](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-options-general.html)
* [Configuring HTTP to HTTPS redirection](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/configuring-https-httpredirect.html)
* [AWS SDK for Python (Boto3)](https://aws.amazon.com/sdk-for-python/)
* [ElasticBeanstalk - Boto 3 Docs](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/elasticbeanstalk.html)
