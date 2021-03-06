[ansible tips]: https://ansible-tips-and-tricks.readthedocs.io/en/latest/ansible/install
[terraform]: https://terraform.io
[terragrunt]: https://github.com/gruntwork-io/terragrunt
[oci]: https://cloud.oracle.com/cloud-infrastructure
[oci provider]: https://github.com/oracle/terraform-provider-oci/releases
[Kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/

# Using this Repo

## Prerequisites

* Download and install [Ansible][ansible tips] **2.4** or higher (`brew install ansible`)
* Download and install [Terraform][terraform] (v0.10.3 or later)
* Download and install [Terragrunt][terragrunt] **0.13.2** or higher (`brew install terragrunt`)
* Download and install the [OCI Terraform Provider][oci provider] (v2.0.0 or later)
* Create an Terraform configuration file at  `~/.terraformrc` that specifies the path to the OCI provider:
  ```
  providers {
    oci = "<path_to_provider_binary>/terraform-provider-oci"
  }
  ```
* Ensure you have [Kubectl][Kubectl] installed if you plan to interact with the cluster locally
  
## Seeding Ansible Vault

If you are forking this repo and want to use it to manage your own live environments, you'll need to:
- Generate some password, to be kept secret and shared among your development team only.
- Regenerate the `./scripts/ansible-vault-challenge.txt` file, which is used in this project's tooling before
encrypting a new managed environment:

```
export ANSIBLE_VAULT_PASSWORD_FILE=/tmp/vault-password
echo $VAULT_PASSWORD > /tmp/vault-password
echo "challenge accepted" > scripts/ansible-vault-challenge.txt
ansible-vault encrypt scripts/ansible-vault-challenge.txt
git add scripts/ansible-vault-challenge.txt
``` 

From this point on, the tooling will enforce that ANSIBLE_VAULT_PASSWORD_FILE is set and contains the correct
password, when dealing with managed environments.

## Setting up a Unmanaged Environment

The following script will create an "unmanaged" environment (i.e. a personal environment just for you 
that won't be checked into Git):

```
python ./scripts/create_env.py my-sandbox --managed false 
```

A number of parameters must be provided to this script, such as OCI tenancy/compartment/user details. 
See the script `--help` for usage. If not specified on the command line, the script will prompt for all required parameters.  

Additionally, a preferences file (default: `~/.k8s/config`) can be used to specify parameters.

```
python ./scripts/create_env.py my-sandbox --prefs /tmp/.k8s/config
```

Here is a sample of the preferences file:

```
[K8S]
managed=false
user_ocid=ocid1.user.oc1..aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
fingerprint=aa:aa:aa:aa:aa:aa:aa:aa:aa:aa:aa:aa:aa:aa:aa:aa
private_key_file=/tmp/odx-sre-api_key.pem
tenancy_ocid=ocid1.tenancy.oc1..aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
compartment_ocid=ocid1.compartment.oc1..aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
region=us-ashburn-1
k8s_master_shape=VM.Standard1.2
k8s_worker_shape=VM.Standard1.2
etcd_shape=VM.Standard1.2
k8s_masters=1,0,0
k8s_workers=0,1,0
etcds=1,1,1
k8s_master_lb_enabled=false
```

### Options To Create_Env

Create_env requires a comma-separated list of the number of K8s master nodes to start in each AD.  A value > 0
must be specified for at least one AD.  For example, specifying 2 masters in AD2 and 1 in AD3:

```
python ./scripts/create_env.py my-sandbox --managed false --k8s_masters 0,2,1 ...
```

Similarly for K8s worker nodes:

```
python ./scripts/create_env.py my-sandbox --managed false --k8s_masters 1,0,0 --k8s_workers 1,3,2
```

A similar comma-separated list is used to specify dedicated Etcd nodes as well:  

```
python ./scripts/create_env.py my-sandbox --managed false --k8s_masters 1,0,0 --k8s_workers 1,3,2 --etcds 1,1,1
```

Specifying no dedicated Etcd nodes will result in Etcds getting colocated with the K8s master nodes:

```
python ./scripts/create_env.py my-sandbox --managed false --k8s_masters 1,0,0 --k8s_workers 1,3,2 --etcds 0,0,0
```

It's possible to create a cluster consisting of only a master node, and to then make the master schedulable
by Kubernetes:

```
python ./scripts/create_env.py my-sandbox --managed false --k8s_masters 1,0,0 --k8s_workers 0,0,0 --etcds 0,0,0
export KUBECONFIG=`pwd`/envs/my-sandbox/files/kubeconfig
kubectl taint nodes --all node-role.kubernetes.io/master-
```

An OCI load balancer can be placed in front of the K8S master nodes, and used in the local kubeconfig file:

```
python ./scripts/create_env.py my-sandbox --managed false --k8s_master_lb_enabled true --k8s_master_lb_shape 100Mbps ...
```

An iSCSI volume can be attached to each K8S worker, for use as the workers' local Docker storage:

```
python ./scripts/create_env.py my-sandbox --managed false --worker_iscsi_volume_create true ...
```

## Setting up a Managed Environment

* First, configure your ansible-vault environment. You'll need to know the secret VAULT_PASSWORD to do this:

```
export ANSIBLE_VAULT_PASSWORD_FILE=/tmp/vault-password
echo $VAULT_PASSWORD > /tmp/vault-password
``` 

Use the `./scripts/create_env.py` script with the **--managed** option, like:

```
python ./scripts/create_env.py prod --managed 
```

Notes:
* After Ansible deployment is complete, the script will encrypt sensitive environment files, commit the
environment's files, create a Git branch, and instruct you to create an MR with the changes.

Upon completion of the script, you'll see a message like this:

```
Environment files have been committed to the local branch dsimone/create-prod-env. Proceed by pushing this branch and creating an MR.
```

At this point, push the branch and create and MR to commit the changes to master.

Alternatively, you can choose to skip branch creation by passing in `--skip_branch`.  A new commit will still be 
created for the new environment's files, but no branch will be created.  One such use for this option is
when you are creating many managed environments in one sitting, and you want to handle branch creation yourself.

## Rolling Out Ansible Changes to Managed Environments

Let's say we are rolling out a change to the `prod` environment:

* First, configure your ansible-vault environment.  You'll need to know the secret VAULT_PASSWORD to do this:
 
```
export ANSIBLE_VAULT_PASSWORD_FILE=/tmp/vault-password
echo $VAULT_PASSWORD > /tmp/vault-password
``` 

#### Terraform Part

**Note** - Use extreme care when running Terraform updates!!!

* Decrypt the certs for the environment:

```
python scripts/decrypt_env.py some-env
```

* `cd envs/sandbox`
* Run ```terragrunt plan -state `pwd`/terraform.tfstate``` to see the changes you're about to apply.
* If the plan looks good, run:

```
terragrunt apply -state=`pwd`/terraform.tfstate
```

#### Ansible Part

* The following takes care of decrypting the environment's files, dynamically populating the Ansible 
inventory and SSH private key, and deploying via Ansible:

```
python ./scripts/ansible_deploy_env.py some-env
```

Or, to run just a specific tag, for example:

```
python ./scripts/ansible_deploy_env.py some-env -tags foo
```
