# Purpose:
To create infrastructure using Terraform and run a Jenkins pipeline deploy a retail banking application to two separate EC2 instances using Jenkins agents. Previously, using the Jenkins user, we created a script to ssh into an instance and install and run the files necessary to deploy the application. Essentially, we manually created a Jenkins agent and coded its job. 

We are also exploring redundancy, by deploying two instances of the application. In the case, one application goes down, traffic can be routed to the second instance.

# Steps to Production:
## 1. Create A Key Pair
### Purpose: 
Creating and associating a key pair with an instance will allow you to ssh into another instance, created with the same key.
### Issues:
Terraform doesn't allow you to create key pairs in the main.tf file, so I had to create it using the GUI.

## 2. Create Infrastructure Using Terraform
### Purpose:
Instead of creating infrastructure manually, you can use Terraform. Therefore, you are able to reuse your main.tf files and edit them for future deployment and stay informed about the environment via the state file.
### Process:
I configured a VPC, 2 availability zones with a public subnet in each. One EC2 instance in us-east-1a with Jenkins installed on it with the following packages:

Software-properties-common, which helps manage different repositories or sources of packages
add-apt-repository -y ppa:deadsnakes/ppa, which allows us to install multiple versions of python
python3.7 and python3.7-venv, which installs python3.7 and the virtual environment package. 


Then I set up 2 instances in us-east-1b and installed default-jre, software-properties-common, sudo add-apt-repository -y ppa:deadsnakes/ppa, python3.7, python3.7-venv. These packages are similar to the packages installed in the first instance and also include installing Java in order to connect and run the agents. 
A route table that connected the instances in the public subnets to the internet gateway. The security group associated with the instances had ports open for 22 - ssh, 8080 - Jenkins, 8000 - gunicorn.

### Issues:

My instance running Terraform had no more space or CPU left, so I had to vertically scale my instance to get more storage and CPU. Moving forward, installing CloudWatch on this instance and creating a alarm for CPU and disk storage would have let me know earlier that I needed to upgrade the server.

While adding the Jenkins script to the user_data for the first instance was successful, the two other scripts did not work on the other instances. I resolved it by updating the repository and entering each command separately.

### Optimization:
I could have create separate security groups for the Jenkins controller and the Jenkins agents. The latter only needs ports 22 and 8000 open. Similarly, the Jenkins server just needs 22 and 8080. Having additional ports open makes our instances vulnerable to attacks.

## 3. Create Jenkins Agents
### Purpose: 
Instead of running the entire build on the main controller, you can divide the pipeline stages into agents launched on instances. Therefore, you are able to assign instance types with the resources necessary to execute that pipeline stage. Using nodes allows you to deploy an application across numerous instances more efficiently. Instead of running the entire pipeline, it runs certain stages - saving resources and improving security. Agents are more secure because they are accessed via ssh, requiring a private key. The Jenkins controller is open to all traffic from the internet.

### Process:
I created two nodes called awsDeploy and awsDeploy2 for the Jenkins controller to ssh into and only build out the stages specified for that agent. Then, I inputted the information like the username, host ip, and key to create a ssh connection via the Jenkins graphical user interface. Also, be sure to select keep the agent online since it will deploy our application beyond the build.

## 4. Create Repository and Update Jenkinsfile
I copied the remote repository to my local device via git clone. Then, created a second branch and edited the jenkins file to apply the clean and build stages to second node awsDeploy2 in addition to the first node, awsDeploy.
Next, I git add . , git commit -m “updated jenkinsfile”, git switch main, git merge second, and git push to Github

## 5. Run Jenkins Pipeline
I installed 'pipeline keep running' to keep the retail banking application deployed after the build. 

### Console Output
The build and test stages were executed via the built-in node on the Jenkins controller. The ‘build’ stage is creating a virtual environmentJenkins used git to clone the repository and fetch the latest commits. The build stage created a virtual environment using python3, activated that test environment, installed python's package manager and all of the packages in the requirements.txt that are necessary for the application to run. The test stage activated the test environment, executed tests, and saved the results in test-reports/results.xml.

Then, the nodes ran the 'Clean' and 'Deploy' stages. I ran the commands in the ‘Clean’ stage and found that the script searched the processes for gunicorn and if the PID was a number other than 0, meaning the process was active, then it killed the process. Plainly, it kills any running gunicorn processes. However, when I ran the command there was more than one gunicorn process, so this script would only rid of one using the head -n 1 command. Though, maybe killing one process, kills them all.

In 'Deploy' stage, a python virtual environment was created and activated and the repository was cloned onto the second EC2 instance. Within the application files directory, pip installed the packages necessary to deploy the application and gunicorn. Then, it ran the database.py and load_data.py files, which created an SQLite database and populated it with account information. Finally, it started the gunicorn server.


Here is the retail banking application on the two instances:
<img width="1440" alt="agent1" src="https://github.com/nalDaniels/Deployment5.1/assets/135375665/f946d026-c08a-4728-8091-957ef7c57e36">

<img width="1438" alt="agent2" src="https://github.com/nalDaniels/Deployment5.1/assets/135375665/bd5ca68e-4479-47d4-951c-4e3f727bf0c8">

# Optimization:
I would improve this deployment by placing the Jenkins server with Jenkins agents in a separate VPC, so internet traffic doesn’t accidentally hit the Jenkins server and have access to code. Then, I would ssh and run a deployment script to instances in an availability zones in a separate VPC. This ensures that traffic coming to the router does not have an opportunity to hit the Jenkins server or agents. The Jenkins agents are only set up when there is new build. This will also help with resource management and contnetion since instances are not running builds and the retail banking application. However, since we are running both, I would install CloudWatch and monitor the CPU, memory, and storage. 


To make the application more available, I would add a third availability zone to deploy the application into. This will also create redundancy if one AZ goes down, users will be sent to another. We would also need a load balancer to distribute traffic to the applications.

# Resources:
Please find my system design for this deployment here:
https://github.com/nalDaniels/Deployment5.1/blob/main/SystemDesign.md



