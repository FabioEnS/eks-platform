# EKS Platform with GitOps

A Kubernetes platform on AWS, built with Terraform and managed through GitOps.
EKS Platform with GitOps
A hands-on project that builds a production-style Kubernetes platform on AWS using Terraform, tests it, and tears it down to keep costs under control.
Status: Core platform built, tested, and destroyed. GitOps (Argo CD) is now built.
What this project is and why I built it
Kubernetes is the industry-standard way to run containerized applications. Amazon EKS (Elastic Kubernetes Service) is AWS's managed version, where AWS runs the control plane and I manage the worker machines. I built this to show I can stand up a complete, repeatable cloud platform from code, not by clicking in the console.
Everything is written in Terraform, so the whole platform can be created with one command and removed with another. That matters for cost: an EKS cluster bills about $0.10 per hour plus worker machines and a NAT gateway, so I built it, tested it, and destroyed it in the same session.
What was built
I built the platform in small steps. Each step was planned, applied, committed, and pushed to GitHub before moving on.
1. Provider and Terraform setup. The AWS provider (version ~> 5.0) in us-east-1. Credentials come from the environment, not from the code, so no profile name is stored in the repo.
2. VPC and subnets. A new VPC (10.0.0.0/16) with four subnets across two availability zones: public 10.0.1.0/24 and 10.0.2.0/24, private 10.0.3.0/24 and 10.0.4.0/24. I used a fresh VPC instead of reusing the landing zone's so this project stands on its own.
3. Internet gateway and public route table. Gives the public subnets a path to the internet.
4. NAT gateway, Elastic IP, and private route table. Lets machines in private subnets reach out to the internet (for example to pull container images) without being reachable from outside. This is the piece that costs about $1 per day, so it was created last and destroyed first.
5. IAM roles. One role for the EKS cluster (eks-platform-cluster-role) and one for the worker nodes (eks-platform-node-role), each with the standard AWS-managed policies attached.
6. EKS cluster. Named eks-platform, spanning all four subnets.
7. Managed node group. Named eks-platform-nodes, using t3.small machines, desired size 2, minimum 1, maximum 2, placed in the private subnets.
How the pieces fit together
• The cluster is the control plane. Think of it as the manager of a lunchroom: it decides what runs where.
• The nodes are the worker machines (the lunch tables) where the applications actually run.
• The IAM roles are permission slips. The cluster role lets EKS manage AWS resources on my behalf. The node role lets the worker machines join the cluster and pull images.
• The private subnets keep the workers hidden from the internet. The NAT gateway lets them make outbound calls only. The public subnets and internet gateway are the front door for traffic that is meant to come in.
Connecting to the cluster
I installed kubectl (the command-line tool for talking to Kubernetes) with winget, then ran:
aws eks update-kubeconfig --region us-east-1 --name eks-platform
kubectl get nodes
The result showed two nodes in Ready state, which confirmed the platform was working.
Issue: kubectl not recognized after install
Right after installing kubectl, VS Code's terminal said the command did not exist. The cause was a cached PATH: terminals only read the list of installed tools when they start. Closing and fully reopening VS Code fixed it. Lesson: after installing a command-line tool, restart the whole editor, not just the terminal tab.
Testing: a sample app and self-healing
To prove the platform actually runs workloads, I deployed a stand-in app, the stock nginx web server, with two copies (replicas):
kubectl create deployment hello-web --image=nginx --replicas=2
Then I deleted one of the running pods on purpose and watched Kubernetes notice and start a replacement automatically. This is self-healing: I told Kubernetes the desired state (two copies), and it keeps the real state matching.
Cost notes and teardown
Item
Approximate cost
EKS cluster
$0.10 per hour
Two t3.small nodes
about $0.04 per hour combined
NAT gateway plus Elastic IP
about $1 per day
VPC, subnets, route tables, IAM roles
free
I decided to skip GitOps for this session rather than leave the cluster running while I set it up. I deleted the sample deployment, then ran terraform destroy from the terraform folder, which removed all 22 resources cleanly.
Lesson learned: verify cleanup against the real account
terraform destroy said everything was gone, and for this project it was. But I did not stop there. I ran a stray-resource check against the actual AWS account:
aws ec2 describe-nat-gateways --region us-east-1
aws ec2 describe-addresses --region us-east-1
aws eks list-clusters --region us-east-1
That check found a NAT gateway and Elastic IP still running that were not from this project. They were leftovers in the landing-zone production environment, quietly costing about $1 per day. I traced them using the VPC address range (10.2.0.0/16 is production), removed them with a targeted destroy, and confirmed the account had zero NAT gateways and zero Elastic IPs afterward.
Terraform only knows about what it is tracking. The real account is the source of truth, so I now run a stray-resource check at the end of every session. The full story is in the landing-zone README.
What's next
• Add GitOps with Argo CD so the cluster's apps are driven from Git.
• Replace the nginx stand-in with a small custom app.
• Keep the build, test, and destroy pattern so the project never leaves running costs behind.

## GitOps with Argo CD

After building the cluster, I added GitOps using Argo CD, so the cluster's apps are driven from Git instead of by typing commands.

**What GitOps means:** Git is the single source of truth. I change a file in GitHub, and the cluster updates itself to match. Nobody has to run manual commands on the cluster.

**What I set up:**
- Installed Argo CD into its own namespace on the cluster.
- Logged in to the Argo CD dashboard through a port-forward tunnel on port 8080.
- Added a small app folder to this repo (apps/hello) with a deployment and a service for a stand-in nginx app called hello-web.
- Created an Argo CD application pointing at that folder on the main branch, with automatic sync turned on.

**Result:** Argo CD showed the app as Healthy and Synced, with two pods running.

**GitOps test:** I changed replicas from 2 to 3 in deployment.yaml, committed, and pushed to GitHub. After a refresh in Argo CD, a third pod appeared on its own, without me touching the cluster.

## Issues Hit and How I Fixed Them

**kubectl not recognized after install**
Right after installing kubectl with winget, the VS Code terminal said the command wasn't found. The cause was a cached PATH: the terminal had started before the install, so it didn't know about the new tool. Fix: fully close and reopen VS Code. Lesson: restart the whole editor after installing a command-line tool.

**Leftover NAT gateway from another project**
After tearing down the cluster, I ran a stray-resource check against the real AWS account instead of trusting that the destroy had caught everything. It found a NAT gateway and Elastic IP still running from the landing-zone production environment, costing about $1 a day. I removed them with a targeted destroy and confirmed zero NAT gateways and zero Elastic IPs afterward. Lesson: always verify cleanup against the real account, not just the Terraform output.

**Terraform could not find my AWS credentials**
When I ran terraform plan, it failed with a credentials error about not being able to reach the instance metadata service, even though I had signed in with AWS SSO. The cause was that Terraform did not know which AWS profile to use. Fix: set the AWS_PROFILE environment variable to my SSO profile in the terminal. In PowerShell, that only lasts for that one terminal window, so it has to be set again in each new terminal. Lesson: when the AWS CLI works but Terraform does not, check which profile Terraform is actually using.

### Cost and Teardown

This project costs money while it runs, so I follow the same routine every session: build, test, destroy, check.

**Approximate costs while running:**
- EKS cluster: about $0.10 per hour
- Two t3.small worker nodes: about $0.04 per hour combined
- NAT gateway plus Elastic IP: about $1 per day
- VPC, subnets, route tables, and IAM roles: free

**Routine:**
1. Build the platform with Terraform and test it.
2. Run terraform destroy at the end of the session.
3. Run a stray-resource check against the real AWS account, looking for NAT gateways, Elastic IPs, and EKS clusters, so nothing is left running by mistake.

Because everything is destroyed after each session, the project's running cost stays at a few cents per build.

## CI Pipeline with GitHub Actions
Every push to this repo now runs a GitHub Actions workflow (.github/workflows/terraform.yml) that checks the Terraform code and runs a real terraform plan against AWS. It logs in to AWS using OIDC, so there are no long-lived AWS keys stored in GitHub.
What the pipeline does
1.  Checks out the code.
2.  Sets up Terraform.
3.  Logs in to AWS by assuming the IAM role eks-platform-github-actions in us-east-1 (OIDC, short-lived credentials).
4.  Runs terraform fmt -check, terraform init, terraform validate, and terraform plan in the terraform folder.
The role is read-only (AWS ReadOnlyAccess policy), so the pipeline can look at the account and plan, but it cannot create or change anything. The plan shows everything as new because Terraform state is local, not remote.
How the login is locked down
The role's trust policy only lets this one repository assume it. The condition on the OIDC sub claim is pinned to this repo's numeric IDs:
repo:FabioEnS@345034732/eks-platform@1432591497:*
The role is defined in terraform/github-oidc.tf. It reuses the account's existing GitHub OIDC provider through a Terraform data block instead of creating a new one.
Issues Hit and How I Fixed Them (Pipeline)
1. Slash in a VS Code new-file name created a folder
Symptom: terraform plan kept saying 22 to add instead of 25, and the file showed up red/orange in the editor.
Cause: I typed a file name with a slash in it. VS Code treated the slash as a folder, which left two copies of github-oidc.tf. Terraform read the stale copy in the terraform folder while I was editing the copy in the leftover folder.
Fix: Copied the edited file over the real one with Copy-Item and deleted the leftover folder.
Lesson: Hover over the tab to check the real file path, and use dir or Select-String to see what is actually on disk.
2. Wrong file ending (.cf instead of .tf)
Cause: Terraform ignores any file that does not end in .tf, so the resources in it were silently skipped.
Fix: Renamed the file to end in .tf.
3. PowerShell split my -target flag
Cause: In PowerShell, an unquoted -target=aws_iam_role.x gets split at the dots.
Fix: Put quotes around each target, like -target="aws_iam_role.x".
4. OIDC provider already existed (409 error)
Symptom: 409 EntityAlreadyExists when creating the GitHub OIDC provider.
Cause: AWS allows only one GitHub OIDC provider per account, and the landing zone project had already created it.
Fix: Used a Terraform data block to look up the existing provider instead of creating a new one.
5. Case-sensitive GitHub username
Symptom: The first pipeline runs failed at the AWS login step with Not authorized to perform sts:AssumeRoleWithWebIdentity.
Cause: My GitHub username is spelled FabioEnS, and I had typed FabioENS. The trust policy match is case-sensitive.
Fix: Corrected the spelling. This was only one of several causes, though (see 6 and 7).
6. Missing sts:TagSession permission
Cause: The GitHub login passes session tags (the log said 7 role session tags were used), and the role's trust policy only allowed sts:AssumeRoleWithWebIdentity.
Fix: Added sts:TagSession next to sts:AssumeRoleWithWebIdentity in the trust policy.
7. The OIDC subject uses numeric IDs, not just names
Symptom: With the condition set to repo:* the login worked, but every version with my real repo name failed (exact case, all lowercase, and a wildcard owner).
Cause: GitHub sends the sub claim with numeric IDs attached to the owner and repo, not just their names.
How I found it: I added a temporary workflow step that printed the token's subject. It showed repo:FabioEnS@345034732/eks-platform@1432591497:ref:refs/heads/main.
Fix: Pinned the condition to repo:FabioEnS@345034732/eks-platform@1432591497:*, applied it, and re-ran the pipeline green. Then removed the debug step and pushed.
Lesson: When a trust policy fails and the reason is not obvious, loosen it temporarily on a read-only role, find out what the real value is, then lock it back down. Never leave it loosened.
Status
The pipeline runs green end to end: login, format check, init, validate, and plan. No infrastructure is running, and the stray-resource check (NAT gateways, Elastic IPs, EKS clusters in us-east-1) came back clean.