[![Build Status](https://travis-ci.com/npalm/terraform-aws-gitlab-runner.svg?branch=master)](https://travis-ci.com/npalm/terraform-aws-gitlab-runner)
[![Gitter](https://badges.gitter.im/terraform-aws-gitlab-runner/Lobby.svg)](https://gitter.im/terraform-aws-gitlab-runner/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

# Terraform module for GitLab auto scaling runners on Spot instances

> *NEW*: The runner will register itself automatically to GitLab. No need to register the runner first, see also the [examples]

This repo contains a Terraform module and examples to run a [GitLab CI multi runner](https://docs.gitlab.com/runner/) on AWS Spot instances. See the blog post at [040code](https://040code.github.io/2017/12/09/runners-on-the-spot/) for a detailed description of the setup.

![GitLab Runners](https://github.com/npalm/assets/raw/master/images/2017-12-06_gitlab-multi-runner-aws.png)

The setup is based on the blog post: [Auto scale GitLab CI runners and save 90% on EC2 costs](https://about.gitlab.com/2017/11/23/autoscale-ci-runners/) The created runner will have by default a shared cache in S3 and logging is streamed to CloudWatch. The cache in S3 will expire in X days, see configuration. The logging can be disabled. The post mentioned that you have to register the the runner before running the Terraform scripts. Since version 3+ this is not needed anymore you can pass the runner configuration including the runner registration token.

Besides the auto scaling option (docker+machine executor) the docker executor is supported as well for a single node.

## Prerequisites

### Terraform

Ensure you have Terraform installed, see `.terraform-version` for the used version. A handy tool to mange your Terraform version is [tfenv](https://github.com/kamatama41/tfenv).

On mac simple install tfenv using brew.

```sh
brew install tfenv
```

Next install a Terraform version.

```sh
tfenv install <version>
```

### AWS

To run the Terraform scripts you need to have AWS keys.
Example file:

```sh
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
```

### Service linked roles

The gitlab runner EC2 instance needs the following service linked roles:

- AWSServiceRoleForAutoScaling
- AWSServiceRoleForEC2Spot

By default the EC2 instance is allowed to create the roles, by setting the option `allow_iam_service_linked_role_creation` to `false` you can deny the creation of roles by the instance. In that case you have to ensure the roles exists. You can create them manually or via terraform.

```hcl
resource "aws_iam_service_linked_role" "spot" {
  aws_service_name = "spot.amazonaws.com"
}

resource "aws_iam_service_linked_role" "autoscaling" {
  aws_service_name = "autoscaling.amazonaws.com"
}
```

### Configuration GitLab runner token

By default the runner is registered the first time. In previous version this was a manual process. The manual process is still supported but will be removed in future releases. The runner token will be stored in the parameter store. See [example](examples/runner-pre-registered/) for more details.

To register the runner automatically set the variable `gitlab_runner_registration_config["token"]` which you can find in your GitLab project, group or global settings. For a generic runner you can find the token in the admin section. By default the runner will be locked to project, not run untagged. Below an example of the configuration map.

```
  gitlab_runner_registration_config = {
    registration_token = "<registration token>"
    tag_list           = "<your tags, comma separated"
    description        = "<some description>"
    locked_to_project  = "true"
    run_untagged       = "false"
    maximum_timeout    = "3600"
  }
```

For migration to the new setup simply add the runner token to the parameter store. Once the runner is started it will lookup required values in the parameter store. If the value is null a new runner will be created.

```
# set the following variables, look up the variables in your Terraform config.
# see your Terraform variables to fill in the vars below.
aws-region=<${var.aws_region}>
token=<runner-token-see-your-gitlab-runner>
parameter-name=<${var.environment}>-<${var.secure_parameter_store_runner_token_key}>

aws ssm put-parameter --overwrite --type SecureString  --name "${parameter-name}" --value ${token} --region "${aws-region}"
```
Once you have created the parameter, you have to remove the variable `runners_token` from your config. Then next time your gitlab runner instance is created it look up the token from the parameter store.

Finally the runner still support the manual runner creation, no changes are required. Please keep in mind that this setup will be removed in future releases.


## Usage

### Configuration

Update the variables in `terraform.tfvars` to your needs and add the following variables, see previous step for how to obtain the token.

```hcl
runner_name  = "NAME_OF_YOUR_RUNNER"
gitlab_url   = "GITLAB_URL"
runner_token = "RUNNER_TOKEN"
```

The base image used to host the GitLab Runner agent is the latest available Amazon Linux HVM EBS AMI. In previous version of the module an hard coded list of AMI per region was available. This list is replaced by a search filter to find the latest AMI. By setting the filter for example to `amzn-ami-hvm-2018.03.0.20180622-x86_64-ebs` you can lock the version of the AMI.


### Usage module.

```hcl
module "runner" {
  source = "npalm/gitlab-runner/aws"
  version = "3.0.0"

  aws_region      = "${var.aws_region}"
  environment     = "${var.environment}"
  ssh_public_key  = "${file("${var.ssh_key_file}")}"

  vpc_id                   = "${module.vpc.vpc_id}"
  subnet_ids_gitlab_runner = "${module.vpc.private_subnets}"
  subnet_id_runners        = "${element(module.vpc.private_subnets, 0)}"

  runners_name       = "my-spot-runner"
  runners_gitlab_url = "https://www.gitlab.com"

  gitlab_runner_registration_config = {
    registration_token = "<YOUR_TOKEN>"
    tag_list           = "docker_spot_runner"
    description        = "Docker AWS Spot runner"
    locked_to_project  = "true"
    run_untagged       = "false"
    maximum_timeout    = "3600"
  }

  runners_off_peak_timezone   = "Europe/Amsterdam"
  runners_off_peak_idle_count = 0
  runners_off_peak_idle_time  = 60

  # working 9 to 5 :)
  runners_off_peak_periods = "[\"* * 0-9,17-23 * * mon-fri *\", \"* * * * * sat,sun *\"]"
}
```


## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| allow_iam_service_linked_role_creation | Boolean used to control attaching the policy to a runner instance to create service linked roles. | string | `true` | no |
| ami_filter | List of maps used to create the AMI filter for the Gitlab runner agent AMI. Currently Amazon Linux 2 `amzn2-ami-hvm-2.0.????????-x86_64-ebs` looks to *not* be working for this configuration. | list | `<list>` | no |
| ami_owners | The list of owners used to select the AMI of Gitlab runner agent instances. | list | `<list>` | no |
| aws_region | AWS region. | string | - | yes |
| aws_zone | AWS availability zone (typically 'a', 'b', or 'c'). | string | `a` | no |
| cache_bucket_prefix | Prefix for s3 cache bucket name. | string | `` | no |
| cache_expiration_days | Number of days before cache objects expires. | string | `1` | no |
| cache_shared | Enables cache sharing between runners, false by default. | string | `false` | no |
| create_runners_iam_instance_profile | Boolean to control the creation of the runners IAM instance profile | string | `true` | no |
| docker_machine_instance_type | Instance type used for the instances hosting docker-machine. | string | `m4.large` | no |
| docker_machine_options | List of additional options for the docker machine config. Each element of this list must be a key=value pair. E.g. '["amazonec2-zone=a"]' | list | `<list>` | no |
| docker_machine_spot_price_bid | Spot price bid. | string | `0.04` | no |
| docker_machine_user | Username of the user used to create the spot instances that host docker-machine. | string | `docker-machine` | no |
| docker_machine_version | Version of docker-machine. | string | `0.16.1` | no |
| enable_cloudwatch_logging | Boolean used to enable or disable the CloudWatch logging. | string | `true` | no |
| enable_manage_gitlab_token | Manage the GitLab token in SSM, if `true` the token SSM parameter will be manged by terraform, a destroy removes the parameter for the runner token. When `false` the token will not be manged by terraform. The runner process will still store the token in SSM but a terraform destroy will not remove the token. | string | `true` | no |
| environment | A name that identifies the environment, used as prefix and for tagging. | string | - | yes |
| gitlab_runner_registration_config | Configuration used to register the runner. See the README for an example, or reference the examples in the examples directory of this repo. | map | `<map>` | no |
| gitlab_runner_version | Version of the GitLab runner. | string | `11.8.0` | no |
| instance_role_json | Docker machine runner instance override policy, expected to be in JSON format. | string | `` | no |
| instance_role_runner_json | Instance role json for the docker machine runners to override the default. | string | `` | no |
| instance_type | Instance type used for the GitLab runner. | string | `t2.micro` | no |
| runners_concurrent | Concurrent value for the runners, will be used in the runner config.toml. | string | `10` | no |
| runners_executor | The executor to use. Currently supports `docker+machine` or `docker`. | string | `docker+machine` | no |
| runners_gitlab_url | URL of the GitLab instance to connect to. | string | - | yes |
| runners_iam_instance_profile_name | IAM instance profile name of the runners, will be used in the runner config.toml | string | `` | no |
| runners_idle_count | Idle count of the runners, will be used in the runner config.toml. | string | `0` | no |
| runners_idle_time | Idle time of the runners, will be used in the runner config.toml. | string | `600` | no |
| runners_image | Image to run builds, will be used in the runner config.toml | string | `docker:18.03.1-ce` | no |
| runners_limit | Limit for the runners, will be used in the runner config.toml. | string | `0` | no |
| runners_monitoring | Enable detailed cloudwatch monitoring for spot instances. | string | `false` | no |
| runners_name | Name of the runner, will be used in the runner config.toml. | string | - | yes |
| runners_off_peak_idle_count | Off peak idle count of the runners, will be used in the runner config.toml. | string | `0` | no |
| runners_off_peak_idle_time | Off peak idle time of the runners, will be used in the runner config.toml. | string | `0` | no |
| runners_off_peak_periods | Off peak periods of the runners, will be used in the runner config.toml. | string | `` | no |
| runners_off_peak_timezone | Off peak idle time zone of the runners, will be used in the runner config.toml. | string | `` | no |
| runners_output_limit | Sets the maximum build log size in kilobytes, by default set to 4096 (4MB) | string | `4096` | no |
| runners_post_build_script | Commands to be executed on the Runner just after executing the build, but before executing after_script. | string | `` | no |
| runners_pre_build_script | Script to execute in the pipeline just before the build, will be used in the runner config.toml | string | `` | no |
| runners_pre_clone_script | Commands to be executed on the Runner before cloning the Git repository. this can be used to adjust the Git client configuration first, for example. | string | `` | no |
| runners_privilled | Runners will run in privileged mode, will be used in the runner config.toml | string | `true` | no |
| runners_request_concurrency | Limit number of concurrent requests for new jobs from GitLab (default 1) | string | `1` | no |
| runners_root_size | Runner instance root size in GB. | string | `16` | no |
| runners_token | Token for the runner, will be used in the runner config.toml. | string | `__REPLACED_BY_USER_DATA__` | no |
| runners_use_private_address | Restrict runners to the use of a private IP address | string | `true` | no |
| secure_parameter_store_runner_token_key | The key name used store the Gitlab runner token in Secure Paramater Store | string | `runner-token` | no |
| ssh_public_key | Public SSH key used for the GitLab runner EC2 instance. | string | - | yes |
| subnet_id_runners | List of subnets used for hosting the gitlab-runners. | string | - | yes |
| subnet_ids_gitlab_runner | Subnet used for hosting the GitLab runner. | list | - | yes |
| tags | Map of tags that will be added to created resources. By default resources will be taggen with name and environemnt. | map | `<map>` | no |
| userdata_post_install | User-data script snippet to insert after GitLab runner install | string | `` | no |
| userdata_pre_install | User-data script snippet to insert before GitLab runner install | string | `` | no |
| vpc_id | The target VPC for the docker-machine and runner instances. | string | - | yes |

## Outputs

| Name | Description |
|------|-------------|
| runner_agent_role | ARN of the rule used for the ec2 instance for the GitLab runner agent. |
| runner_as_group_name | Name of the autoscaling group for the gitlab-runner instance |
| runner_cache_bucket_arn | ARN of the S3 for the build cache. |
| runner_role | ARN of the rule used for the docker machine runners. |

## Example

An example is provided, execute the following steps to run the sample. Ensure your AWS and Terraform environment is set up correctly. All commands below are supposed to be run inside the directory `example`.

### AWS keys

Keys are generated by Terraform and stored in a directory `generated` in the example directory.

### Configure GitLab

Register a new runner:

```sh
docker run -it --rm gitlab/gitlab-runner register
```

Once done, lookup the token in GitLab and update the `terraform.tfvars` file.

## Create runner

Run `terraform init` to initialise Terraform. Next you can run `terraform plan` to inspect the resources that will be created.

To create the runner run:

```sh
terraform apply
```

To destroy runner:

```sh
terraform destroy
```
