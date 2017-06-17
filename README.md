# Overview
This will get you set up to start using Terraform for your project. What you choose to do with the underlaying infrastructure is your choice. I have used this Terraform code, combined with Ansible for provisioning data onto server, for over two years.

- - - -
# Getting started before you can actually get started

Make sure you have terraform, aws, and jq installed. I am an Ansible guy, so I will be leveraging that to gpg verify/install Terraform for me. You can copy/paste this entire block.

    sudo pip install --upgrade awscli ansible
    sudo yum install -y jq
	# https://github.com/pgporada/ansible-role-terraform
	# ansible-galaxy installs roles to /etc/ansible/roles/
	sudo ansible-galaxy install --force pgporada.terraform
	cat << EOF > playbook.yml
	---
	- hosts: localhost
	  connection: local
	  vars:
	    - vagrant_version: 0.9.8
	  roles:
	    - pgporada.terraform
	...
	EOF
	ansible-playbook playbook.yml -b -K

Verify you have your specified version of Terraform

	terraform version

I will not be working in the default AWS region so I must specify a --profile for `aws` commands. If you happen to be working out of the default region though, you can run

    export AWS_PROFILE=default

To start, we need an S3 bucket to store state. This bucket will NOT be managed by Terraform. This is the classic chicken and egg problem.

    aws s3 mb s3://${AWS_BUCKET} --profile=${AWS_PROFILE}

The next thing we'll need is a KMS key to encrypt state when it is stored in S3.

    aws kms create-key --description="Terraform state encryption/decryption" --tags TagKey=Name,TagValue="terraform-state-key" --profile ${AWS_PROFILE} | jq -r .KeyMetadata.KeyId;

If you have a bunch of keys and you forget what the KeyId was, you can loop through them as follows

    for i in $(aws kms list-keys --profile ${AWS_PROFILE} | jq -r .Keys[].KeyId); do \
        aws kms describe-key --key-id $i --profile ${AWS_PROFILE} | grep -C10 Terraform | jq -r .KeyMetadata.KeyId; \
    done

# Actually getting started
- - - -
### Overview
We are going to be creating a VPC with X number of public/private subnets. We will also be creating a public/private route53 domain.

### VPC
Grab my `terraform-vpc` project from github! I encourage you to walk through the code. You can email me or leave an issue if you have questions.

    git clone https://github.com/pgporada/terraform-vpc.git

Fill out the environment tfvars data

    cd terraform-vpc
    vim environments/example.tfvars

Configure initial terraform setup for the terraform-vpc project. I believe you should only ever have to do this once per project.

    # Example vars
    ##############
    # AWS_REGION=us-east-2
    # AWS_STATE_BUCKET=somebucket
    # AWS_PROFILE=example
    # AWS_KMS_ARN=arn:kms:666sdnfsdkljfn/345345/4nldfngkndfgfk
    # ENVIRONMENT=prod

	terraform init \
        -backend-config="region=${AWS_REGION}" \
        -backend-config="bucket=${AWS_STATE_BUCKET}" \
        -backend-config="profile=${AWS_PROFILE}" \
        -backend-config="key=terraform/vpc/${ENVIRONMENT}.tfstate" \
        -backend-config="encrypt=1" \
        -backend-config="acl=private" \
        -backend-config="kms_key_id=${AWS_KMS_ARN}"

To see the result of the `terraform init`, you can cat out the state file.

	cat .terraform/terraform.tfstate

Run a plan. A plan shows you what terraform thinks it should do based on the changes you've made to your `.tf` files and what terraform has stored in its state file.

    terraform plan -var-file environments/example/example.tfvars

After eyeballing the plan and feeling confident, you can run an apply. THIS WILL BUILD STUFF!

    terraform apply -var-file environments/example/example.tfvars

To see that nothing else needs to be built, we can run a plan again.

    terraform plan -var-file environments/example/example.tfvars
