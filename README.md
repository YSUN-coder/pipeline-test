# AppSync template for CloudFormation with TaskCat

We need use CloudFormation template to automatically deploy an AppSync API to communicate with IoT edge device using the AWS Security Tokens.

## Tools

1. Taskcat
   [taskcat](https://github.com/aws-quickstart/taskcat) is a tool that tests AWS CloudFormation templates. It deploys your AWS CloudFormation template in multiple AWS Regions and generates a report with a pass/fail grade for each region.

2. Visual Studio with CloudFormation Linter
   The AWS CloudFormation linter provides additional static analysis beyond what is provided by running `aws cloudformation validate-template`. You can use it as a pre-commit hook for syntax/grammar correctness validation. In Visual Studio Code, you can use the tool via the linter extension "CloudFormation Linter" (kddejong.vscode-cfn-lint). If you are using the JetBrains editor PyCharm, there is an `aws-cloudformation` plugin for the linter.

## Prerequistes

If this is your first-time deployment, you need to follow this Prerequistes section.

### AWS IAM

Create a role to delegate permissions to an AWS service.


Configure the IAM role named 'test_with_formation' that the credentials provider assumes on behalf of your device. Attach the following trust policy (trust relationships) to the role.
```
{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Principal": {"Service": "cloudformation.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }
}
```

Add the following policy to the IAM entity that needs to create the service role. This policy allows you to create a service role for the specified service and with a specific name. You can then attach managed or inline policies to that role. This statement —“Resource”: “*”— will allow you to create any service role for any service, and then attach managed or inline policies to that role named 'test_with_formation'.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:AttachRolePolicy",
                "iam:CreateRole",
                "iam:PutRolePolicy"
            ],
            "Resource": "*"
        }
    ]
}
```

The user who performs the operation of passing the service role has ```iam:PassRole``` permission to authorize this action. You also should add permission for the ```iam:GetRole``` action to allow the user to retrieve information about the specified role. Create the following policy to grant iam:PassRole and iam:GetRole permissions. In my case, I named this policy 'test_passrole_to_formation_permission' and attached it to my AWS account.

```
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": [
        "iam:GetRole",
        "iam:PassRole"
    ],
    "Resource": "arn:aws:iam::<aws_user_id>:role/test_with_formation"
  }
}
```
### AWS CLI Installation and Configuration
If you have not setup the AWS CLI in your Local work platform(your laptop), you should follow the instructions in [Installing AWS CLI version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) and [CLI configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html). After these steps, you will have the default user in ~/.aws/credentials, which will use for providing access to your AWS account.

## Automation Installation
### Install Local templates test
#### Create virtual environment

It's a best practice to use virtual environment when working with python projects.

TaskCat requires python3. Therefore, we will create a virtual environment with python3 as default interpreter before installing TaskCat.

Run following commands, to create a virtual environment and use python3 as default interpreter for virtual environment.

```
virtualenv -p /usr/bin/python36 vpy36
source vpy36/bin/activate
```

Run `python --version`, and you should see the output as below:

<pre>
$ python --version
Python 3.6.x
</pre>

#### Install TaskCat

TaskCat can be installed with Docker or by using pip. We will use pip for this workshop.

From your terminal, run the following command.

`pip install taskcat`

This will install the latest TaskCat version 9.

After the installation is finished, run `taskcat --version` to ensure that TaskCat version 9 is installed correctly.

#### Prepare tests

TaskCat requires a project configuration file to configure details about the project and define tests.

This file is created at **\<PROJECT_ROOT\>/.taskcat.yml**.

There are 2 configuration sections in this file as shown below:

1. Project configuration (project)
2. Test configuration (tests)

<pre>
project:
  name: test-appsync-project
  regions:
    - ca-central-1
tests:
  appsync-test:
    parameters:
      APIName: "simpleAPI"
      AvailabilityZones: "ca-central-1"
    template: ./templates/appsyncApi.template
</pre>

#### Running tests for Templates
Now that we have taskcat configuration file created, let's run TaskCat to execute our tests.

Run the following command in your terminal window:

```
cd test-cloudformation-project

taskcat test run
```

### Install CI/CD with AWS Pipeline
In AWS CI/CD, we expect the CloudFormation stack configures CodePipeline to run with every GitHub change. We need a pipeline to fully automate the process of running TaskCat tests as part of a deployment pipeline in AWS CodePipeline and CodeBuild. To build your CI/CD environment for testing AWS CloudFormation templates on AWS, follow the instructions in the deployment guide. The GitHub repository contains a configuration file, a parameters file, and the AWS CloudFormation templates you want to test.

#### pipeline-taskcat configuration (pipeline)

For the process of building the Stack from github, we need add configurations about CodeBuild as follows:
```
CodeBuildSetup:
      Source:
        Type: GITHUB
        Location: !Sub https://github.com/${GitHubUser}/${GitHubRepo}.git
```

Therefore, we need the GitHubToken of the project User in https://github.com/<GitHub_User>/appsync-automation to give the AWS Pipeline ne the access to GitHub Repository. To apply the GitHubToken, please go to https://github.com/settings/tokens to get or generate your github token.

In this step, Personal access tokens can only be used for HTTPS Git operations. If your repository uses an SSH remote URL, you will need to [switch the remote from SSH to HTTPS](https://docs.github.com/en/github/getting-started-with-github/getting-started-with-git/managing-remote-repositories#switching-remote-urls-from-ssh-to-https).

In order for CodePipeline to use GitHub as a source provider it needs your GitHub personal access token, and store your GitHub Personal Access Token in AWS Secrets Manager. Here are the steps:
- Go to the AWS Secrets Manager Console, and change the region to ca-central-1.
- Click Secrets and click the Store a new secret button.
- Click on the Other type of secrets radio button.
- Click on the Plaintext tab and enter the GitHub token value in the text area. You got this token in the last step.
- Leave the Select the encryption key dropdown with the DefaultEncryptionKey option selected.
- Click the Next button.
- Enter 'github/personal-access-token' for the Secret name and description on the Secret name and description page and click Next.
- On the Configure automatic rotation page, select the Disable automatic rotation radio button.
- Click the Next button.
- On the Review page, click the Store button.
Then, we could use the secrest in pipeline-taskcat.yml as follows:
```
GitHubToken:
    Default: '{{resolve:secretsmanager:github/personal-access-token:SecretString}}'
```


#### Files
* buildspec.yml
This file installs and configures Python, pip, and TaskCat. It also runs the TaskCat tests against the CloudFormation templates listed in the .taskcat.yml file. The buildspec.yml file is configured to run as part of a CodeBuild project defined in the pipeline-taskcat.yml CloudFormation template. In this template CodePipeline is configured to execute this CodeBuild project.

* pipeline-taskcat.yml
This CloudFormation template provisions two CodeBuild projects, IAM Permissions, S3 Buckets, and a deployment pipeline in AWS CodePipeline. Once this CloudFormation stack is successfully launched, a pipeline will run in which CodeBuild will run CloudFormation tests in TaskCat, create a static website in S3, and copy the TaskCat dashboard files to this website.

There are two CodeBuild projects in this template. CodeBuildTest runs the TaskCat tests and is configured to run from the CodePipeline resource. CodeBuildWebsite copies the files generated and stored in the taskcat_outputs folder to an S3 bucket and then configures the S3 bucket to be a public static website. CodePipeline is also configured to run this CodeBuild project.

This CloudFormation stack configures CodePipeline to run with every GitHub change. If you’d rather run these tests on a scheduled basis, you will need to make changes to the configuration.

#### Run the CI/CD pipeline

```
$ cd <ROOT-PROJECT-PATH>
$ aws cloudformation create-stack --stack-name <your_stack_name> --capabilities CAPABILITY_NAMED_IAM --disable-rollback --template-body file:///<abs-path-to-root-repo>/appsync-automation/templates/appsyncApi.template --parameters ParameterKey=APIName,ParameterValue=<your_api_name>
```
<your_stack_name>: the name that you want to give to the new stack, e.g. basic-AppSync-test-sylvia


## Reference
* https://github.com/aws-quickstart/quickstart-workshop/blob/develop/content/70_testing/1_local_test.md
