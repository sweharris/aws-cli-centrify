# aws-cli-centrify
bash script to allow Centrify 2FA to be used to get access tokens to use the aws CLI tools

[Centrify](https://www.centrify.com) provides a number of tools in their
suite.  The one we're interested in, here, is the cloud user portal.
In this we can configure single sign on connectivity to third party
applications, e.g. using your AD kerberos ticket.  Typically this
uses an IDentity Prodiver (IDP) function, and may return a SAML assertion
that is then passed to the application.

One such application is the [AWS Console](https://community.centrify.com/t5/TechBlog/How-To-Integrate-AWS-Console-with-Centrify-Identity-Service-SAML/ba-p/28296).

The idea, here, is that you authenticate to Centrify (perhaps using 2FA)
and select the _role_ you want.  Centrify will eventually redirect you
to the Amazon console with the appropriate role.   

Note that I've said "role"; you don't have a user in your AWS account any more,
which makes it difficult to create a `~/.aws/credentials` entry to use the
[CLI tools](https://aws.amazon.com/cli/).

To address this, Centrify have a python script to help gain temporary
credentials.  I'm not sure if it's publically released, but they
have [documentation](https://docs.centrify.com/en/centrify/appref/index.html#page/cloudhelp/a-f/saas_appref_amazonsaml_python.html).

This script will talk to the Centrify APIs to perform the authentication
steps, then call the application tile to get the SAML assertion, which can
then be passed to the AWS `STS` endpoint to get tokens.  These are then
written to the `credentials` file.

Unfortunately this requires Python 3.5 and a number of extra modules.  If,
like me, you use CentOS 7 (or RedHat or Oracle) this is a problem.  It
_is_ possible to use the software collections library and some extra `pip
install` commands to meet the requirements... but it's not simple.

So I decided to reimplement it in `bash` instead.

This script has a number of dependencies:

*    `jq`  - available in EPEL for RedHat
*    `base64` - to read the SAML response
*    `curl` - to talk to Centrify
*    `aws`  - the Standard aws CLI tool to get the identity tokens

The output is a number of `export` variables, making this suitable
for `eval`.

It makes it very easy to create a function wrapper

## Example function:

    centrify_2fa()
    {
      eval $( CENTRIFY_ENDPOINT="example.centrify.com" CENTRIFY_APPKEY="12345678-1234-1234-1234-123456789012" centrify ${1:-example.user@example.com} )
    }

## In use:

    % centrify_2fa
    Using example.centrify.com for user example.user@example.com
    Enter Password:
    Select authentication method:
       1: "Mobile Authenticator"
       2: "Email... @example.com"
       3: "SMS... XXX-1234"
    Enter choice: 2
    Performing 'Out of Band' authentication, waiting for completion
    .......
    Select role:
       1: arn:aws:iam::123456789901:role/aws_centrify
       2: arn:aws:iam::123456789901:role/aws_admin
    Enter choice: 1
    Keys valid until 2018-05-23T21:12:46Z

    % aws sts get-caller-identity
    {
        "Account": "123456789901",
        "UserId": "AROAIHWUHJHJGL23HGSHK:example.user@example.com"
        "Arn": "arn:aws:sts::123456789901:assumed-role/aws_centrify/example.user@example.com"
    }

## License

This code is under GPLv2 or, at your preference, any later version.
