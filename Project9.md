INSTALL AND CONFIGURE JENKINS SERVER
## Step 1 – Install Jenkins server
1. Create an AWS EC2 server based on Ubuntu Server 20.04 LTS and name it "Jenkins"   

![jenkins](./images/01-jenkins.PNG)   

2. Install JDK (since Jenkins is a Java-based application)
```
sudo apt update
sudo apt install default-jdk-headless
```   
![jdk](./images/02-jdk-install.PNG)  

3. Install Jenkins
```
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt-get install jenkins
```   
![install](./images/03-jenkinsinstall.PNG)   

Make sure Jenkins is up and running
```
sudo systemctl status jenkins
```   
![running](./images/04-jenkins%20running.PNG)

4. By default Jenkins server uses TCP port 8080 – open it by creating a new Inbound Rule in your EC2 Security Group


5. Perform initial Jenkins setup.
From your browser access http://<Jenkins-Server-Public-IP-Address-or-Public-DNS-Name>:8080

You will be prompted to provide a default admin password  

![jenkins browser](./images/05-jenkins%20browser.PNG)


Retrieve it from your server:
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
![pass](./images/06-password.PNG)   

Then you will be asked which plugings to install – choose suggested plugins.



Once plugins installation is done – create an admin user and you will get your Jenkins server address.   

![url](./images/08-jenurl.PNG)  


The installation is completed!



## Step 2 – Configure Jenkins to retrieve source codes from GitHub using Webhooks
In this part, you will learn how to configure a simple Jenkins job/project (these two terms can be used interchangeably). This job will be triggered by GitHub webhooks and will execute a ‘build’ task to retrieve codes from GitHub and store it locally on Jenkins server.

1. Enable webhooks in your GitHub repository settings. enter `http://<jenkins adress>/github-webhook/` in the payload url

![webhooks](./images/09-webhooks.PNG)  

2. Go to Jenkins web console, click "New Item" and create a "Freestyle project"

![tooling](./images/10-freestyle.PNG)

To connect your GitHub repository, you will need to provide its URL, you can copy from the repository itself

**note**-enale proxy compactibility in the Configure Global Security setting
![setting](./images/11-enable%20proxy.PNG)  


In configuration of your Jenkins freestyle project choose Git repository, provide there the link to your Tooling GitHub repository and credentials (user/password) so Jenkins could access files in the repository.

![git-add](./images/12-git-add.PNG)  


Save the configuration and let us try to run the build. For now we can only do it manually.
Click "Build Now" button, if you have configured everything correctly, the build will be successfull and you will see it under #1

![build1](./images/13-build.PNG)

You can open the build and check in "Console Output" if it has run successfully.

If so – congratulations! You have just made your very first Jenkins build!

But this build does not produce anything and it runs only when we trigger it manually. Let us fix it.

3. Click "Configure" your job/project and add these two configurations
Configure triggering the job from GitHub webhook:

![config](./images/14a-config.PNG)

Configure "Post-build Actions" to archive all the files – files resulted from a build are called "artifacts". 

![config2](./images/14b-config.PNG)


Now, go ahead and make some change in any file in your GitHub repository (e.g. README.MD file) and push the changes to the master branch.

You will see that a new build has been launched automatically (by webhook) and you can see its results – artifacts, saved on Jenkins server.

![artifacts](./images/15-artifacts.PNG)

You have now configured an automated Jenkins job that receives files from GitHub by webhook trigger (this method is considered as ‘push’ because the changes are being ‘pushed’ and files transfer is initiated by GitHub). There are also other methods: trigger one job (downstreadm) from another (upstream), poll GitHub periodically and others.

By default, the artifacts are stored on Jenkins server locally
```
ls /var/lib/jenkins/jobs/tooling_github/builds/<build_number>/archive/
```

CONFIGURE JENKINS TO COPY FILES TO NFS SERVER VIA SSH
## Step 3 – Configure Jenkins to copy files to NFS server via SSH
Now we have our artifacts saved locally on Jenkins server, the next step is to copy them to our NFS server to `/mnt/apps` directory.

Jenkins is a highly extendable application and there are 1400+ plugins available. We will need a plugin that is called "Publish Over SSH".

1. Install "Publish Over SSH" plugin.
On main dashboard select "Manage Jenkins" and choose "Manage Plugins" menu item.

On "Available" tab search for "Publish Over SSH" plugin and install it



2. Configure the job/project to copy artifacts over to NFS server.
On main dashboard select "Manage Jenkins" and choose "Configure System" menu item.

Scroll down to Publish over SSH plugin configuration section and configure it to be able to connect to your NFS server:

1. Provide a private key (content of .pem file that you use to connect to NFS server via SSH/Putty)
2. Arbitrary name
3. Hostname – can be `private IP address` of your NFS server
4. Username – `ec2-user` (since NFS server is based on EC2 with RHEL 8)
5. Remote directory – `/mnt/apps` since our Web Servers use it as a mointing point to retrieve files from the NFS server
Test the configuration and make sure the connection returns `Success`. Remember, that TCP port 22 on NFS server must be open to receive SSH connections.

![test](./images/16-test-conf.PNG)

Save the configuration, open your Jenkins job/project configuration page and add another one "Post-build Action"


Configure it to send all files probuced by the build into our previouslys define remote directory. In our case we want to copy all files and directories – so we use `**`.
If you want to apply some particular pattern to define which files to send – use this syntax.

![send](./images/17-send%20build%20artifacts.PNG)

**note**: change permissions on NFS server
```
sudo chown -R nobody: /mnt/apps
sudo chmod -R 777 /mnt/apps
```   

Save this configuration and go ahead, change something in `README.MD` file in your GitHub Tooling repository.

Webhook will trigger a new job and in the "Console Output" of the job you will find something like this:
```
SSH: Transferred 25 file(s)
Finished: SUCCESS
```
![trans](./images/18-transfered.PNG)  

To make sure that the files in `/mnt/apps` have been updated – connect via SSH/Putty to your NFS server and check README.MD file
```
cat /mnt/apps/README.md
```
If you see the changes you had previously made in your GitHub – the job works as expected.

![result](./images/19-result.PNG)

Congratulations!
You have just implemented your first Continous Integration solution using Jenkins CI. Watch out for advanced CI configurations in upcoming projects.

