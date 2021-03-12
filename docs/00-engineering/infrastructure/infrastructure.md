# Infrastructure Best Practices

Bixal utilizes infrastructure as code (IaC) for services that support it. IaC provides reproducibility and allows for ephemeral environments. Different technologies exists to accomplish this task from [Terraform](https://www.terraform.io/), [Ansible](https://www.ansible.com/), [Chef](https://www.chef.io/), [Puppet](https://puppet.com/), or [SaltStack](https://www.saltstack.com/) are common technologies that we've run across.

To document all the best practices of each different technology you encounter is out of the scope of this document. The goal of this page is to cover the major security concerns and serve as an introduction how to build and maintain infrastructure.

# FedRAMP

## What is FedRAMP

FedRAMP is an program authorized by the OMB to streamline the 'Authority to Operate' process and simplify setting up technology systems and fulfilling [FISMA](https://www.dhs.gov/cisa/federal-information-security-modernization-act) requirements. It's very verbose with some 800+ controls related to the security and procedures around ownership of technology at the federal level. To sum up what it addresses:

* Who has access to the system?
* What roles are there?
* What PII is contained within the system?
* What are the impacts if the system is not available?
* What is the plan if the system needs to be recovered from physical or technology disasters.
* How is the system backed up?
* Who is securing and updating technology in the system?

FedRAMP lets us use a platform like Acquia, and not have to be responsible for the physical hardware and only be responsible
for the application and user management.

# Network Architecture

Projects should always take network architecture into account when designing servers and infrastructure. All Cloud Service Providers (CSP's) have the ability to create public, and private subnet's. By utilizing this architecture services can remain private and more protected, and services such as bastion hosts may be used to harden access to private services. For more information on this type of architecture you can review AWS's [reference documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Scenario2.html). While this documentation is AWS specific, the concepts apply across CSP's.

# Amazon Web Services

Currently Bixal utilizes AWS with it's cloud hosting provider.

## IAM and Role Based Access

All CSP's, which have a fedRAMP authorization have have strong identity and role based management. It is imperative that teams follow a policy of least privilege and never simply grant full access to a service. These controls, along with use of multi factor authentication are of the highest priority when dealing access to environments. Credentials should **NEVER** be shared all of the following services will allow for unique identifiers and credentials to be created for users and services.

### Accessing Cross-Account Roles

Generally access is given to a role by using AWS Security Token Service. You will have a user which has been given a policy to [assume](https://aws.amazon.com/premiumsupport/knowledge-center/iam-assume-role-cli/) a role. In order to be able to assume a role, you must have [multi-factor authentication](https://aws.amazon.com/iam/features/mfa/) set up. This only works with the Virtual device MFA, not with Yubikey U2F, in the CLI. Both Virtual devices and U2F can be used in the console.
 
Once you have a user account, with MFA and password setup you can create an [access key](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey) or use the one provided to you. 

#### Console

In order to assume a role in the AWS console that exists on an account that is not the one that you have a user on, do the following:

1. Open the link given to you with your user credentials.
1. Use the provided username and password to sign in.
1. In the upper right corner, next to the bell, click on your username.
1. Then click on "Switch Role", then "Switch Role" again at the following screen.
1. Enter the account number and role name provided in the role ARN. More details [here](#role-arn-breakdown).
1. Click "Switch Role" again. You will be redirected to the account your role is on.


##### Assign MFA for yourself by:
1. Clicking on your username next to the Bell icon in the top right bar
1. Clicking on "My Security Credentials"
1. "Manage MFA device" and add your preferred device
1. Sign out of AWS and back in using your MFA device

#### CLI

Once you have an access key you can install the [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html). To configure the aws cli with a main sign on and cross account role with mfa, create a ~/.aws/credentials file. This file should look something like this:

```text
[default]
aws_access_key_id = MYACCESSKEY
aws_secret_access_key = MYSECRETACCESSKEY
[projectname]
role_arn = arn:aws:iam::9999999999:role/developer-role
source_profile = default
mfa_serial = arn:aws:iam::111111111:mfa/my-username
region = us-east-1
```
Here is what you will need to `<change>` from the example:
```text
[default]
aws_access_key_id = <change>
aws_secret_access_key = <change>
[<change>]
role_arn = arn:aws:iam::<change>:role/<change>
source_profile = default
mfa_serial = arn:aws:iam::<change>:mfa/<change>
region = <change if needed>
```
To access resources under this assumed role:

```sh
aws --profile projectname <awscli subcommand>
```
For example, to list the S3 buckets in myCoolProject:
```sh
aws --profile myCoolProjectProfileName s3 ls
```
where `myCoolProjectProfileName` is the name of the block similar to `projectname` above.

If you prefer to not use the `--profile` command option you may:
```sh
export AWS_PROFILE=myCoolProjectProfileName
aws s3 ls
```
and `aws s3 ls` will be executed using the profile that you just exported.

For each of these command examples, you will be prompted for your Virtual MFA device's 6 digit code. You will not be prompted for this again unless you change terminals or your role expires. In either case, supplying the new code will allow you to assume the role again.

#### Role ARN Breakdown

When you are granted access to a role that allows cross-account access the ARN will look like this: `arn:aws:iam::000000000000:role/role-name`. This includes two parts that are important for assuming that role through the console:

1. Account number: `000000000000`
1. Role name: `role-name`

## Gov Cloud vs Public Regions

The primary differences between Gov Cloud and a General Public regions is staffing, and availability of services and FedRAMP authorizations. Gov Cloud has FedRAMP high along with some related DoD impact level certifications. It achieves this by staffing only US Citizens and a reduction in overall services where they cannot be compliant to the standard.

What this means as an architecture perspective is the following:

* Does our application need FedRAMP (or DoD impact level) high?
* Does our application utilize services not available in Gov Cloud?
* Does our application need to be served, or interconnected to global audience? (E.g - a USAID project which is us funded by USAID but may have a global audience).
