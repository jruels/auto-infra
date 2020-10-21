# Automate deployment of resources using CICD pipeline

In this tutorial you will be using Jenkins, Terraform and Ansible to automation deployment of infrastructure.
This is a first step in building a CICD pipeline.

In this tutorial, you'll complete these tasks:

  * Create a basic Jenkins project.
  * Set up environment variables so Terraform and Ansible can create infrastructure.
  * Create a Jenkins build job and GitHub webhook for automated builds.
  * Test the CI/CD pipeline to deploy infrastructure through automation.

## Prerequisites
### GitHub
Create a free GitHub account if you do not already have one.   
https://github.com/join

Fork the following GitHub repository for the sample code - [https://github.com/jruels/tf-labs](https://github.com/jruels/tf-labs)  

To fork the repo to your own GitHub account, select the **Fork** button in the top right-hand corner.

Setup Jenkins Webhook for automated builds. 

Open GitHub repository settings in browser: 
https://github.com/<your_github_account/tf-labs/settings/hooks

Click "Add webhook" 
Fill in the Payload URL field with:
`http://<jenkins-ip>/github-webhook/`

Leave defaults and click "Add webhook"

Generate Personal Access Token to authenticate from Jenkins to GitHub.
1. In a browser visit: [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. Log in to GitHub if prompted
3. Click "Generate new token"
	1. Name token
	2. Select "repo" and "user:email" scopes
4. At the bottom of page click "Generate token"
Make sure you save your token because it will not be shown again.

Clone the repo to your development system. Make sure you use the URL of your fork when cloning this repo:

```
git clone https://github.com/<your-github-account>/tf-labs
```

Change to the directory containing the pipeline code:

```
cd tf-labs
```

## Create required service account 
We must configure Jenkins, Terraform and Ansible so they have access to our Google Cloud infrastructure. 

1. Create service account using `gcloud`
Type the following to create a new service account named `jenkins`
```
gcloud iam service-accounts create jenkins --display-name jenkins
```
2. Store the service account email address and your current Google Cloud project ID in environment variables for use in later commands:
```
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:jenkins" --format='value(email)')
export PROJECT=$(gcloud info --format='value(config.project)')
```
3. Bind the following roles to your service account: 
```
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.instanceAdmin.v1 \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.networkAdmin \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.securityAdmin \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/iam.serviceAccountActor \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/storage.objectViewer \
    --member serviceAccount:$SA_EMAIL
```
### Encode the service account key
Now that you've granted the service account the appropriate permissions, you need to create and encode its key. This key will be used when configuring Jenkins and Terraform.

1. Create the key file: 
```
gcloud iam service-accounts keys create jenkins-sa.json --iam-account $SA_EMAIL
```
2. In Cloud Shell, run the following command to encode the contents of the key.
```
base64 jenkins-sa.json
```

Copy the output and save it for use later in the lab. 

## Terraform State
Previously when we’ve run Terraform, you’ll notice that some state files get written to our local directory. Terraform uses these to make its graph of changes every time it runs. So if we run Terraform in a pipeline, where does it write its state file?
The answer is to store the state in a bucket in the project itself. Then anyone can run Terraform in the pipeline and the remote state is centrally stored and shared. Terraform will also manage locking of the state to make sure conflicting jobs aren’t run at the same time.
Run the following in Cloud Shell to create a Google Cloud Storage bucket for our state:
```
gsutil mb -c regional -l us-west2 gs://$DEVSHELL_PROJECT_ID-tfstate
```


### Update Project 
A lot of the files being used for this lab do not support environment variables. They need to be updated to match the GCP projectID.

After creating the bucket we need to update the `backends.tf` file to use the new bucket.  

It should look like this: 
```yaml
terraform {
  backend "gcs" {
    bucket = "<your projectID here>-tfstate"
    credentials = "./creds/jenkins-sa.json"
  }
}
```

Create the `creds` directory and copy the `jenkins-sa.json` file to it.
```
mkdir creds 
cp jenkins-sa.json creds/
```

Now reinitialize Terraform to read the backend configuration 
```
terraform init -reconfigure
```

Ansible is configured to use a dynamic inventory and requires the inventory `yml` file be updated with your account's projectID

In `tf.gcp.yml` update the `projects` field so it looks like below. 
```yaml
plugin: gcp_compute
projects:
  - <Your projectID here>
auth_kind: serviceaccount
service_account_file: creds/jenkins-sa.json
filters: []
keyed_groups:
  - key: labels
    prefix: label
groups:
  dev: "'web-instance' in name"
```

Update the `Jenkinsfile` so that it also has the correct projectID set. 
```groovy
environment {
  SVC_ACCOUNT_KEY = credentials('jenkins-gcp')
  PROJECT_ID = "<Your projectID Here>"
  DEFAULT_LOCAL_TMP = 'tmp/'
  ANSIBLE_USER = 'ubuntu'
  HOME='/tmp'
  }
```
After updating the above files, commit them and push to your forked repository.
```
git add .
git commit -m 'update project id'
git push
```

## Install Ansible
An Ansible binary is required on the Jenkins server. 

Log into the Jenkins instance through SSH and install Ansible using `pip3`
```
ssh -i ~/.ssh/tf-key bitnami@<jenkins-vm-ip>
sudo pip3 install ansible requests google-auth
```


## Jenkins
### Initial Setup
In this step we are going to upgrade Jenkins to the latest version. After the upgrade is complete we are going to upgrade and install some plugins. 

1. Click "Manage Jenkins" on the left hand slide.
2. You will see an alert that says "New version of Jenkins". To install it click the button labeled "or Upgrade Automatically" on the ride side of the screen.
3. Select "Restart Jenkins when installation is complete and no jobs are running" to complete upgrade.
4. Update all plugins by clicking "Manage plugins", scrolling to the bottom, clicking "Compatible" and then "Download now and install after restart".
5. Install plugins by click on Updates at the top of the "Manage Plugins" page.  Search for and install the following:
	1. Blue Ocean
	2. Google Compute Engine
	3. GitHub Integration

### Credentials
Now we need to configure Jenkins credentials to allow Terraform and Ansible to run. 

Create new credential with base64 encoded service account `json` policy file. 
1. Click "Manage Jenkins"
2. Click "Manage Credentials
3. On the Credentials screen under "Domains" click "(global)"
4. On the left hand slide click "Add Credentials"
5. Under "Kind" select "Secret text"
6. In the 'Secret' field paste the base64 encoded string for `jenkins-sa.json`
7. Set the ID to `jenkins-gcp`, this will be called from the `Jenkinsfile`

#### Copy SSH public key to Jenkins VM
We need to copy the SSH public key from Cloud Shell to Jenkins VM. 
1. In the Cloud Shell run the following: 
```
scp -i ~/.ssh/tf-key ~/.ssh/tf-key.pub bitnami@<VM IP>:
``` 

2. Log into the Jenkins VM and move the key to `/opt/bitnami/jenkins/jenkins_home`
```
sudo mv tf-key.pub /opt/bitnami/jenkins/jenkins_home/
sudo chown jenkins.root /opt/bitnami/jenkins/jenkins_home/tf-key.pub
sudo chmod 600 /opt/bitnami/jenkins/jenkins_home/tf-key.pub
```
Create another credential for our SSH private key so that Ansible can connect to our provisioned virtual machine.
1. Click "Manage Jenkins"
2. Click "Manage Credentials
3. On the Credentials screen under "Domains" click "(global)"
4. On the left hand slide click "Add Credentials"
5. Under "Kind" select "SSH username with Private Key"
6. Set ID to `cicd-ssh-key`
7. Set Username to `ubuntu`
8. Provide a description
9. Click "Private Key", "Add" and paste the private key from Cloud Shell `~/.ssh/tf-key`
10. Leave "Passphrase" blank.

## Create Pipeline 
On the left side of Jenkins click "Blue Ocean". This will load a new page with the Blue Ocean UI. The first thing it will prompt you to do is create a new pipeline.  
1. Select GitHub as the location for your code
2. Provide the Personal Access Token 
3. Choose your organization 
4. Choose the `tf-labs` repository
5. Click "Create pipeline" and cross your fingers! 


## Pipeline execution 
Jenkins will now kick off the pipeline and if everything is successful you should see it create a new VM and install Docker on it. 





  
