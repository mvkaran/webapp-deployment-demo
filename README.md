# Web App Deployment on the Cloud

This tutorial shows how to deploy a simple Python Django application on the Cloud using AWS and Cloudflare.

Author: MV Karan | https://karan.be | mv at karan.be

---

### Prerequisites

This tutorial assumes you have a basic knowledge of the following:

- Linux OS & Commands
- Git
- Python



The required system/tools are:

- Debian-based OS (Like Ubuntu, above 16.04)
- Git

  - [Installation & Setup Instructions](https://www.digitalocean.com/community/tutorials/how-to-install-git-on-ubuntu-18-04-quickstart)
- Python 3

  - [Installation](https://phoenixnap.com/kb/how-to-install-python-3-ubuntu)



Accounts you would need:

- [GitHub](https://github.com)
- [AWS](https://aws.amazon.com) 
- [Cloudflare](https://cloudflare.com)



#### Demo Application

The [demo application](https://github.com/app-generator/django-dashboard-adminator) that will be used in this tutorial is an open-source dashboard admin panel built using Django framework on Python

---

## Step-by-step Tutorial

Contents:

- [Web App Deployment on the Cloud](#web-app-deployment-on-the-cloud)
    - [Prerequisites](#prerequisites)
      
    - [Demo Application](#demo-application)
  - [Step-by-step Tutorial](#step-by-step-tutorial)
    - [Preparing the demo application](#preparing-the-demo-application)
      - [1. Create a copy of the repository](#1-create-a-copy-of-the-repository)
      - [2. Run the application locally](#2-run-the-application-locally)
      - [3. Commit the changes to your GitHub repository](#3-commit-the-changes-to-your-github-repository)
    - [Setting up an AWS Account](#setting-up-an-aws-account)
    - [Creating cloud infrastructure on AWS](#creating-cloud-infrastructure-on-aws)
      - [1. Create an EC2 instance](#1-create-an-ec2-instance)
      - [2. Login to your EC2 Instance](#2-login-to-your-ec2-instance)
      - [3. Setup your EC2 instance with required tools](#3-setup-your-ec2-instance-with-required-tools)
    - [Manually Deploy your application](#manually-deploy-your-application)
      - [1. Setup Git on EC2](#1-setup-git-on-ec2)
      
      - [2. Download your application](#2-download-your-application)
      
      - [3. Test your application](#3-test-your-application)
      
      - [4. Add hostname to environment file](#4-add-hostname-to-environment-file)
      
      - [5. Run Gunicorn server](#5-run-gunicorn-server)
      
      - [6. Create a socket for gunicorn](#6-create-a-socket-for-gunicorn)
      
      - [7. Create a service for gunicorn](#7-create-a-service-for-gunicorn)
      
      - [8. Enabling and testing the service](#8-enabling-and-testing-the-service)
      
      - [9. Configuring NGINX server](#9-configuring-nginx-server)
      
      - [10. Register DNS on Cloudflare (Optional)](#10-register-dns-on-cloudflare-optional)
      
      - [11. Use domain name (Optional)](#11-use-domain-name-optional)
      
        

### Preparing the demo application

---

#### 1. Create a copy of the repository

We need to first create a clone (i.e, a copy) of the repository. We do this by opening up the terminal and typing the following command.

```bash
git clone --bare https://github.com/app-generator/django-dashboard-adminator.git
```



Go to your GitHub account and create a new repository with the name ``django-dashboard-adminator`` . Select the visibility as ``Public`` . Instructions for creating a new repository can be found [here](https://help.github.com/en/github/getting-started-with-github/create-a-repo).



Go back to your terminal and change directory to the downloaded folder.

```bash
cd django-dashboard-adminator.git
```

Create a mirror (copy) of this repo in your own GitHub account, where ``<username>`` is your GitHub username.

```bash
git push --mirror https://github.com/<username>/django-dashboard-adminator.git
```

Go back to your GitHub account and verify that all the files have been uploaded. 

Let's the delete the local 'bare' repository and get our newly uploaded repository.

To delete the bare repository:

```bash
cd ..
rm -rf django-dashboard-adminator.git
```

To downloaded our newly created repository:

```bash
git clone git@github.com:<username>/django-dashboard-adminator.git
```

Change PWD to the application directory

```bash
cd django-dashboard-adminator
```



#### 2. Run the application locally

Let us first test whether the application that we just downloaded runs well locally or not. For this, there are a few installations & setup that we need to do. We'll create an 'installation' script to do this for us.

Create a new file named ``installation.sh`` and open it with VI or nano editor.

```bash
nano installation.sh
```

Add the below lines to the installation script

```bash
virtualenv env --python=python3
source env/bin/activate
pip3 install -r requirements.txt
python3 manage.py makemigrations
python3 manage.py migrate
```

Run the installation script

```bash
bash installation.sh
```

Let's activate our virtual environment

```bash
source env/bin/activate
```

Run the application

```bash
python3 manage.py runserver
```

Open your web browser and navigate to the URL ``http://localhost:8000``

Create a new account and login.

You should now be able to see the dashboard screen!



#### 3. Commit the changes to your GitHub repository

Go back to terminal and press ``Ctrl+C`` to stop the server.

Let's change what are the changes that Git is currently tracking

```bash
git status
```

You can see that Git is currently tracking only the installation script we created. However, we also created a user account which would be saved in the database file. We need to ensure that this change is also tracked.

_Note: Never add your database file to your VCS since this could potentially expose it to the whole world. We are doing it in this case only as an example and for illustration._ 

```bash
git add --force db.sqlite3 installation.sh
```

Let's now commit this to our changes

```bash
git commit -m "Added installation script & tracking database"
```

Let us now 'push' (i.e, upload) this to our repo on GitHub

```bash
git push
```

We are now ready to deploy our application on AWS!



### Setting up an AWS Account

---

Create an account on [AWS](https://aws.amazon.com). This would require a Debit/Credit card and also a working phone number for verification. Once created, we will be using (almost) free resources for the demo application.



### Creating cloud infrastructure on AWS

---

#### 1. Create an EC2 instance

1. In the ``Services`` menu, select ``EC2``
2.  On the EC2 Dashboard, click on the ``Launch instance`` button
3. In the search box, search for ``Ubuntu Server``
4. In the results window, locate the entry with ``Ubuntu Server 16.04 LTS`` with ``Free tier eligible``.
5. Click on the ``Select`` button next to that entry.
6. In the Instance Type, select ``t2.micro`` which is Free tier eligible
7. Click on ``Next: Configure Instance Details``
8. Leave everything as default in the Configure Instance Details page.
9. Click on ``Next: Add Storage``
10. In the Add Storage page, under ``Size (GiB)``, change the size to ``20``. Leave everything else as default.
11. Click on ``Next: Add Tags``. Leave it as it is.
12. Click on ``Next: Configure Security Group``
13. In the ``Security group name``, type ``django-dashboard-adminator-sg``
14. Add the below rules by clicking on the ``Add Rule`` button:
    1. Type: HTTP
    2. Type: Custom TCP Rule
       1. Port Range: 8000-8002
15. Click on ``Review and Launch``
16. Review that you have entered everything correctly. Click on ``Launch``
17. In the Select an existing key pair or create a new key pair, select ``Create a new key pair`` from the dropdown
18. For the Key pair name, enter ``MyEC2KeyPair`` and click on ``Download Key Pair`` 
19. Make a note of where you are saving the key pair as this is essential to access your EC2 instance. Once you have ensured this, move to next step.
20. Click on ``Launch Instances``
21. Click on ``View Instances``

Wait until ``Instance State`` shows as ``running`` 

Make a note of the ``IPv4 Public IP``. This is the IP address of your EC2 instance that you will be using to access your instance and for all other configuration.



#### 2. Login to your EC2 Instance

We will be using SSH to login to the EC2 instance. First, we need to change the permissions of our key pair file that we downloaded. Go to the terminal on your Linux Machine

```bash
cd /path/to/where/you/have/your/pem/file
chmod 600 MyEC2KeyPair.pem
```

Now, we can SSH to our Instance

```bash
ssh -i /path/to/MyEC2KeyPair.pem ubuntu@<ip-address>
```

Where ``/path/to/`` is the absolute path of where you have you stored the MyEC2KeyPair on your system, and ``<ip-address>`` is the IPv4 Public IP of your EC2 instance above.



#### 3. Setup your EC2 instance with required tools

First, lets update aptitude sources

```bash
sudo apt-get update
```

Install NGINX server

```bash
sudo apt-get install nginx
```

Install virtualenv

```bas
sudo apt-get install virtualenv
```



Our EC2 instance is now ready to begin deploying our application.



### Manually Deploy your application

---

#### 1. Setup Git on EC2

You first need to generate an SSH key within the EC2 instance.

```bash
ssh-keygen
```

Change your PWD to the directory where the public key will be saved.

```bash
cd /home/ubuntu/.ssh
```

Show the contents of the public key

```bash
cat id_rsa.pub
```

Copy the entire output you see.

Add this SSH key to your GitHub account using the instructions found [here](https://help.github.com/en/github/authenticating-to-github/adding-a-new-ssh-key-to-your-github-account).



#### 2. Download your application

Change your PWD to your home directory

```bash
cd ~
```

We need to 'clone' our Git repo to our local EC2 instance manually.

```bash
git clone git@github.com:<username>/django-dashboard-adminator.git
```

Change your PWD to the application folder

```bash
cd django-dashboard-adminator
```

Run the installation script that you had created

```bash
bash installation.sh
```



#### 3. Test your application

Activate virtual environment

```bash
source env/bin/activate
```

Run the test server to check if all dependencies are working

```bash
python3 manage.py runserver
```



#### 4. Add hostname to environment file

Open ``.env`` file with nano or VI editor

```bash
nano .env
```

Change the content after ``SERVER=`` with the IP address of the EC2 instance

Save and exit.



#### 5. Run Gunicorn server

Let's now test using the actual server we will be using, to see whether the application runs. 

```bash
gunicorn --bind=0.0.0.0:8001 core.wsgi:application
```

Open your browser and enter the url ``<ip-address>:8001`` where ``<ip-address>`` is the IP address of your EC2 server. 

If you see the login screen of the application, everything is working fine.



#### 6. Create a socket for gunicorn

We need to create a Unix socket for Gunicorn to be able to communicate with Nginx. I have already created this for you. You need to download it as below

```bash
sudo wget https://github.com/mvkaran/webapp-deployment-demo/raw/master/gunicorn.socket -P /etc/systemd/system/
```



#### 7. Create a service for gunicorn

We need to service in order to handle socket connections. I have already created this for you. You need to download it as below:

```bash
sudo wget https://github.com/mvkaran/webapp-deployment-demo/raw/master/gunicorn.service -P /etc/systemd/system/
```



#### 8. Enabling and testing the service

We need to now start and enable the socket

```bash
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

Let's check the status of the socket

```bash
sudo systemctl status gunicorn.socket
```

Let's check the status of the service

```bash
sudo systemctl status gunicorn
```



#### 9. Configuring NGINX server

The Gunicorn server will serve the application through a socket. However, we would need an NGINX server to pass the traffic back and forth between Gunicorn and the user client. For this, we need to create a configuration. 

I have already created this for you. You need to download it as below:

```bash
sudo wget https://github.com/mvkaran/webapp-deployment-demo/raw/master/django-dashboard-adminator -P /etc/nginx/sites-available
```

You need to add your EC2 instance's IP address in this configuration file. Open up the file using VI or Nano editor

```bash
sudo nano /etc/nginx/sites-available/django-dashboard-adminator
```

Replace ``server_name_OR_IP`` with the IP address of the EC2 instance and save it.

We need to now enable the NGINX configuration by creating a symlink.

```bash
sudo ln -s /etc/nginx/sites-available/django-dashboard-adminator /etc/nginx/sites-enabled
```

Let's test the NGINX configuration:

```bash
sudo nginx -t
```

If everything goes well, we can restart the NGINX server

```bash
sudo systemctl restart nginx
```



#### 10. Register DNS on Cloudflare (Optional)

_Note: For this step, you need to have a registered domain name from a domain registrar (like GoDaddy) and have access to modify Nameserver records._

Create a free account on Cloudflare. 

1. Once you have created and logged-in, click on ``+ Add a Site``
2. Enter your domain name that you have registered. Click on ``Add site``
3. Select Free plan and click on ``Confirm plan``
4. In the A record, type the ``IPv4 address`` of your EC2 instance and ``Name`` as ``@``
5. Click on the orange cloud icon to make into into grey.
6. Set the Automatic TTL to 2 minutes
7. Click on ``Add Record`` and then ``Continue``
8. Change the nameservers on your domain registrar, to the ones showed by Cloudflare
9. Click on ``Done, check nameservers`` once you have changed it on your domain registrar



#### 11. Use domain name (Optional)

Once Cloudflare confirms DNS changes, we need to change it in our application's ``.env`` file as well

Open ``.env`` file with nano or VI editor

```bash
nano .env
```

Change the content after ``SERVER=`` with your domain name

Save and exit.

Let's re-start gunicorn so that it picks up these changes

```bash
sudo systemctl restart gunicorn
```

Next, we need to change this in the NGINX configuration file as well

```bash
sudo nano /etc/nginx/sites-available/django-dashboard-adminator
```

Replace the EC2 IP address that you have with your domain name. 

Let's restart NGINX for it to pick up the changes

```bash
sudo systemctl restart nginx
```

