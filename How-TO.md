# How TO

## Request Openshift 4.6 Shared Environment

## Access and explore playbooks repository

We have (1) collections folder, where we define the collections and roles requirements for the execution of the playbooks in this repository.

We also have (1) variables folder, that contains one file per environment, setting the specific variables for stagging and production.  

Finally, we have (1) playbook, called **deploy-to-ocp.yml**, that identifies the environment, ensure the Openshift project exists and using the template, create the resources for the application. 

## Create Ansible Automation Platform resources

Using AAP, we first create the credential type ```Source Control```, to store the private key to access my personal account in GitHub. Previously, I should have [added my public key into my personal account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account), and then I can use the private key to interact with the repository through SSH. This credential will be used to clone and access the repository containing the playbook to deploy our GitOps into Openshift. 

Then, we create the ```Project``` that states where are the playbook that are going to be used by the Job Templates to execute our automation. 

* Project Name: Openshift GitOps
* Source Control Type: Git 
* Source Control Credential: Github SSH Personal
* Source Control URL: git@github.com:AlmendraSalgado/ansible-demo-gitops.git

Using the localhost inventory that came previously created at the installation of the platform, we create a ```Job Template``` that runs the playbook **deploy-to-ocp.yml** and awaits its execution by a webhook that will be trigerred by GitHub.

### GitHub Webhook

Following the methodology of GitOps, our Infrastructure as Code should be in a different repository where the team responsible, will be working on the code following their guidelines. The best practice tells us that they will be working on a Stagging branch and each production release will be do by pull requests into the main branch. 

Then, following these guidelines the Github webhook should be triggered when there is a push in any of the branch, but AAP should be capable to identify and pass it to the playbook the environment it should deploy the code. 

This is done by using the extra variables provided by the Github webhook. The following is an extract of the extra variables the webhook send to AAP.

```yaml
awx_webhook_event_type: push
awx_webhook_event_guid: 9ba52ae8-2f24-11ed-8317-230859a69f52
awx_webhook_event_ref: bbf0e4f7bb84c62d39fac9583adf1a0a69c7e986
awx_webhook_status_api: null
awx_webhook_payload:
  ref: refs/heads/main
  before: bdbd188db218549bb3d9886a2b6bed7973042663
  after: bbf0e4f7bb84c62d39fac9583adf1a0a69c7e986
  repository:
    id: 533922368
    node_id: R_kgDOH9MCQA
    name: fevermap
    full_name: AlmendraSalgado/fevermap
    private: true
    owner:
      name: AlmendraSalgado
      email: AlmendraSalgado@users.noreply.github.com
      login: AlmendraSalgado
```

The variable we should use is ```awx_webhook_payload.repository.default_branch```. Using this variable we could determine which environment should be deployed and we are going to be capable to use just one Job Template for all the environments and using one push event webhook to trigger the automation. Cool ah?

### AAP Job Template creation

As we mentioned before we introduced the Github webhook, we have to create the job template. Before to do that, and in order to be capable to send status to Github after we received the webhook and execute the playbook, we need to create a webhook credential in AAP.

This type of credential is ```GitHub Personal Access Token```, and we previously should have [created a personal access token in GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token). 

Finally to create the job template we will use the following information.

* Name: Deploy to Openshift
* Inventory: localhost
* Project: Openshift GitOps
* Job Type: run
* Playbook: deploy-to-ocp.yml
* Enable Webhook?: True
* Webhook Service: GitHub
* Webhook Credential: GitHub Personal

After saving the template, we can see ```Webhook URL``` and ```Webhook Key``` and [create the webhook in GitHub](https://docs.github.com/en/developers/webhooks-and-events/webhooks/creating-webhooks).

### Playbook Git Clone

To be able to clone the Infrastructure as Code repository, inside of our playbook we use the ansible git module. In this case we will need to use the private key to clone it, but AAP doesn't have by default a way to pass a GitHub credential to the playbook execution. Instead, we have to create a custom credential to add it to the job template following this guide: https://termlen0.github.io/2019/06/08/observations/


