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

## Cost and Teardown

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