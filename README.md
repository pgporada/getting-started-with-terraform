# Overview
This will get you set up to start using Terraform for your project. What you choose to do with the underlaying infrastructure is your choice. I have used this Terraform code, combined with Ansible for provisioning data onto server, for over two years.

- - - -
# Setup

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

Fill out the environment tfvars data

    cd terraform-vpc
    vim environments/example.tfvars

Configure initial terraform setup for the terraform-vpc project. I believe you should only ever have to do this once per project.

	terraform init \
        -backend-config="region=${AWS_REGION}" \
        -backend-config="bucket=${AWS_STATE_BUCKET}" \
        -backend-config="profile=${AWS_PROFILE}" \
        -backend-config="key=terraform/$(BUCKETKEY)/$(ENVIRONMENT).tfstate" \
        -backend-config="encrypt=1" \
        -backend-config="acl=private" \
        -backend-config="kms_key_id=${AWS_KMS_ARN}"

To see the result of the `terraform init`, you can cat out the state file.

	cd terraform-vpc
	cat .terraform/terraform.tfstate

Run a plan

    terraform plan -var-far environments/example/example.tfvars
