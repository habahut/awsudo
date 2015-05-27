# awsudo
sudo-like utility to manage AWS credentials

Ever wish you could easily provide your AWS credentials to applications when
necessary, but not have them sitting around in environment variables when they
aren't needed? Want to avoid putting your credentials in disparate
configuration files for each tool you use?

`awsudo` works just like sudo: stick it before a command to run that command in
a context that has access to your AWS credentials. Say you wrote a great script
to deploy your project, but it needs an AWS API key to work:

    awsudo ./aws_deployment_script

Most tools will attempt to get credentials from [environment variables].
`awsudo` reads the same configuration files as the [AWS CLI], and then runs the
command you specify with those environment variables set.

Do you have multiple accounts? `awsudo` also understands [named profiles]. If a
profile is configured to [assume a role], `awsudo` can do that also, meaning
you can easily use [temporary credentials] with most AWS tools. To use the
profile `deployment`:

    awsudo -u deployment ./aws_deployment_script

  [environment variables]: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-environment
  [AWS CLI]: http://aws.amazon.com/cli/
  [named profiles]: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-multiple-profiles
  [assume a role]: http://docs.aws.amazon.com/cli/latest/userguide/cli-roles.html
  [temporary credentials]: http://docs.aws.amazon.com/STS/latest/UsingSTS/Welcome.html

## Quickstart

Install awsudo, which will also install the AWS CLI if you don't already have
it. Then configure `aws`, which configures `awsudo` because it reads the same
configuration files:

```console
$ pip install git+ssh://git@github.com/paperg/awsudo.git
$ aws configure
AWS Access Key ID [None]: AKIAIXAKX3ABKZACKEDN
AWS Secret Access Key [None]: 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
Default region name [None]: us-east-1
Default output format [None]: table
# configuration files get created in ~/.aws
$ cat ~/.aws/credentials
[default]
aws_access_key_id = AKIAIXAKX3ABKZACKEDN
aws_secret_access_key = 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
$ cat ~/.aws/config
[default]
output = table
region = us-east-1
# and now awsudo will set the relevant environment variables
$ awsudo env | grep AWS
AWS_ACCESS_KEY_ID=AKIAIXAKX3ABKZACKEDN
AWS_DEFAULT_REGION=us-east-1
AWS_SECRET_ACCESS_KEY=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
```

    note -- if you are getting permission denied (publickey) for pip installation try ```sudo pip install git+git://github.com/paperg/awsudo.git```

## Configuring Profiles

If you have roles your account can assume, you can configure profiles so `aws`
and `awsudo` can use them. To configure a profile you need to know the role's
ARN. You can get it by looking at the role in the AWS web management console
under the IAM section:

![screenshot of web console](doc/arn.png)

If you know the account number and the name of the role, then the ARN is just

> arn:aws:iam::*accountNumber*:role/*roleName*

Now add it to a new section in `~/.aws/config` so it looks like the following.
You can name the profile whatever you like -- here I name it "dev":

```
[default]
output = text
region = us-east-1

[profile dev]
role_arn = arn:aws:iam::123456789012:role/developer
source_profile = default
region = us-east-1
```

Now we can test that it works by asking `awsudo` to run `env`, which just
prints all the environment variables, and then grepping for the variables
relevant to AWS:

```console
$ awsudo env | grep AWS
AWS_ACCESS_KEY_ID=AKIAIXAKX3ABKZACKEDN
AWS_DEFAULT_REGION=us-east-1
AWS_SECRET_ACCESS_KEY=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
$ awsudo -u dev | grep AWS
AWS_SESSION_TOKEN=OEmP+u+j0rj0oQxi1WTBKuezy4RMXs1/425V7hM8STV8V+7zYGLxNCnEqSiqEzc/RROR3Iu4exE+lPVWcvMb6Oc3jX4N/XLj2398jzih15K/eZY10fRpp9rYEPry9tWUCua96eHNtFEA2zzmrGD6sl+YAiiScP+JuzPhbpa3j3qDeRem9m6Lvq7Yh2grNl5PBLlu3Nwg4miKa1y7re6lopY/JURGyJlhWCFnpp54h5MKy8LQZRezze3ADkCW2ki5TI7dtKZIoKHiXKKOXcSRTj2ajcVrVFBln/KlO/lGCMeoodO/piIvNZHdo6LINB8qVRC2oif2kmLEZASKRB5ytbASxG2fyzN9rJkhGAgilvigTijmNt1Tf0hJK2tE+zX2eQ==
AWS_ACCESS_KEY_ID=ASIAJBEFTVQMTRQK7Q2A
AWS_DEFAULT_REGION=us-east-1
AWS_SECRET_ACCESS_KEY=l8UJvTqOFJLmhiEFksflzKbhCOCmoVf3vAYkAtTw
```

# awsrotate

This package also includes a utility to easily rotate your AWS API keys. If you
followed the quickstart instructions above, you should have your key stored in
exactly one place: `~/.aws/credentials`.

To rotate that key, simply run:

    awsrotate

This will:

1. Delete any old and inactive keys you have. It will abort if there are any
   active keys besides the one it finds in `~/.aws/credentials`.
2. Create a new key.
3. With the new key (so we are sure it works), deactivate your old key.
4. Update `~/.aws/credentials`.
