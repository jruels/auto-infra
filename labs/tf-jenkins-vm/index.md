# Deploy Jenkins with Terraform 

In this lab you will use Terraform to deploy a Jenkins virtual machine on GCP, which will be used for future CICD Pipelines.

Start by logging in to the [GCP Console](https://console.cloud.google.com) and launching the Cloud Shell by clicking the terminal icon to the right of the search bar.

## Prerequisites


Install the latest version of Terraform
```bash
cd ~
TER_VER=`curl -s https://api.github.com/repos/hashicorp/terraform/releases/latest | grep tag_name | cut -d: -f2 | tr -d \"\,\v | awk '{$1=$1};1'`
wget https://releases.hashicorp.com/terraform/${TER_VER}/terraform_${TER_VER}_linux_amd64.zip
unzip terraform_${TER_VER}_linux_amd64.zip && sudo mv terraform /usr/local/bin/
```

Confirm installation was successful
```
terraform version 
```
Output should be similiar to: 
`terraform v0.14.5`

Enter the working directory created previously.
```
cd $(date +%Y%m%d)
```

## Clone the Terraform repository
```
git clone https://github.com/jruels/tf-jenkins
cd tf-jenkins
```
##
Review the contents of the directory: 
```
sudo apt install -y tree 
tree -L 2 . 
```
You will see the following: 
```
├── main.tf
├── modules
│   ├── terraform-google-kubernetes-engine
│   └── tf-gcp-jenkins
├── providers.tf
├── README.md
└── variables.tf
```

- main.tf: This is the main configuration file for Terraform. In this file you describe how your infrastructure should be configured. 
- providers.tf: Terraform uses plugins called providers that each define and manage a set of resource types. Most providers are associated with a particular cloud or on-premises infrastructure service, allowing Terraform to manage infrastructure objects within that service.
- variables.tf: This file is used to define configurable variables that are passed to Terraform.
- modules: A module is a container for multiple resources that are used together. Modules can be used to create lightweight abstractions, so that you can describe your infrastructure in terms of its architecture, rather than directly in terms of physical objects.

## Initialize Terraform environment
When running a Terraform configuration for the first time you must initialize the environment. This initialization downloads the required plugins, sets up state and confirms Terraform has everything required to run.
```
terraform init 
```

You will see Terraform initialize modules, backend and providers. If everything is successful you will see: 

```
Initializing modules...
- jenkins in modules/tf-gcp-jenkins

..snip

Terraform has been successfully initialized!
```


## Preview the changes Terraform will make
Terraform requires environment details to be able to successfully run. We are going to use Google Cloud Shell's builtin variable `DEVSHELL_PROJECT_ID` to pass our `project_id` to Terraform as a variable.
Use Terraform's plan feature to confirm what changes will be applied: 
```
terraform plan -var project_id=$DEVSHELL_PROJECT_ID -var jenkins_workers_project_id=$DEVSHELL_PROJECT_ID
```
This should show that Terraform is going to create 13 resources. 
- Firewall rules 
- Compute instance 
    - metadata    
- IAM service accounts 
- Password to log into Jenkins

If everything looks good let's apply the changes 
## Terraform apply 
```
terraform apply -var project_id=$DEVSHELL_PROJECT_ID -var jenkins_workers_project_id=$DEVSHELL_PROJECT_ID
```

You should now see Terraform creating the requested resources. Once the configuration has been applied Terraform will display configured `outputs`
```
Outputs:

jenkins__initial_username = user
jenkins_initial_password = password
jenkins_instance_name = jenkins-test
jenkins_instance_public_ip = <Jenkins IP address> 
```

## Confirm resources were created 
Now let's confirm Terraform created what we requested. 
1. In a browser visit the IP address from Terraform output. 
2. Enter the username and password from Terraform output.

You should now be logged into your Jenkins instance! 

## Summary 
Terraform is a powerful tool that is designed to create and manage infrastructure and other resources. In an upcoming lab you will configure a CICD pipeline that utilizes Terraform and Ansible.

