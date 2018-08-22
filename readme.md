![](./images/amazon-lightsail.jpg)

# Deploying and Scalling Applications with Amazon Lightsail

## Lab 1 - Deploying a Standalone MEAN Application
In this lab you'll deploy a MEAN stack application into a single Lightsail instance. This means that both the Express front end and the Mongo database will be running on the same host. You will build a new application instance, configure the MongoDB database, and then test out the application 

### 1.1: Build the MEAN application instance
The first step is create a MEAN stack instance in Lightsail. When creating the instance, you will use a Lightsail launch script to install the application.

1) From the Lightsail console click `Create Instance`

    ![](./images/1-1-1.jpg)

2) Under `Blueprint` choose `MEAN`

    ![](./images/1-1-2.jpg)

3) Click `+Add Launch Script`

    ![](./images/1-1-3.jpg)

4) Paste the script below into the text box

        #! /bin/bash
        sudo /opt/bitnami/ctlscript.sh stop apache
        sudo mv /opt/bitnami/apache2/scripts/ctl.sh /opt/bitnami/apache2/scripts/ctl.sh.disabled

        cd /home/bitnami
        sudo git clone https://github.com/mikegcoleman/todo
        cd /home/bitnami/todo
        sudo npm install --production

        sudo cat <<EOF >> /home/bitnami/todo/.env
        PORT=80
        DB_URL=mongodb://tasks:tasks@localhost:27017/?authMechanism=SCRAM-SHA-1&authSource=tasks
        EOF

    This script does the following:

    * Stops Apache using the control scripts present in the Bitnami MEAN image. Stopping Apache frees up Port 80 so the application can use it. 
    * Clones the application GitHub repo and installs all the dependencies using the Node Package Manager (`npm`)
    * Creates a configuration file that sets the application port (`80`) and the connection string for the database running on the loca lhost (`mongodb://tasks:tasks@localhost:27017/?authMechanism=SCRAM-SHA-1&authSource=tasks`)

5) Name the instance `MEAN`

    ![](./images/1-1-5.jpg)

6) Click `Create`

    ![](./images/1-1-6.jpg)

Once the instance shows a state of running in the Lightsail console, SSH into it either using the built in SSH client or using your own (username: `bitnami`). If you`re unfamiliar with SSH please see this tutorial. 

![](./images/mean-running.jpg)

***Note**: Even though the instance shows a state of running, it may still be executing the startup script, and you won`t be able to connect. If this is the case, give it a couple of minutes and try again.*
    
### 1.2 - Configure the Mongo database

In this section you are going to configure the Mongo database for use with the application. Specifically you are going to add a username and password  (`tasks` for both) for the application database (also called `tasks`). You'll do all this using the Mongo client from the Lightsail command line. 

***Note**: The following steps are performed from the Lightsail instance command line*

1) First log into the mongo client using the following command

        mongo admin --username root -p $(cat ./bitnami_application_password)

    ***Note**: Each Bitnami-based Lightsail instance stores the application password in a file called `bitnami_application_password`. Below we`re redirecting that file into the Mongo client command line.*

2) Create the tasks database by issuing the following command:

        use tasks

3) Add a db admin user to our tasks database by pasting in the lines below into the Mongo client and hitting `enter`

        db.createUser(
            {
                user: "tasks",
                pwd: "tasks",
                roles: [ "dbOwner" ]
            }
        )

4) Close the mongo shell by typing exit

        exit

### 1.3 - Start and test our application

Now that the application is installed and the database configured, the application is ready to be started. 

***Note**: The following steps are performed from the Lightsail instance command line*

1) Change into the application directory

        cd ~/todo

2) Start the application by executing the following command
    
        sudo DEBUG=app:* ./bin/www

    You should see a message such as:

        app:* Listening on port 80 +0ms

3) From the home page of the Lightsail console get the IP address of the MEAN instance and navigate to that address in a web browser.

    ***Note:** If you are not on the Lightsail home page click `Home` in the upper left corner of the Lightsail console*  

    ![](./images/mean-ip.jpg)

You should see the ToDo application running. Add a task or two to make sure it`s working as expected. 

***Note:** You can also check the output in your SSH session to verify everything is working.

## Lab 2 - Scaling the Todo application's web front-end

The first iteration of the application's web front end is not inherently scalable becaue the database and front end are located on the same machine. It would be problematic to add additional database instances whenever additional front-end capacity was needed. 

To fix this issue the front end and database each need to be running on their own instance. In this lab you will create an instance for the database and the web front end. Then you will take a snapshot of the web front end, and deploy two additional web front end instances from that snapshot. Finally, you'll add a loadbalancer in front of the three front end instances. 

### 2.1 - Create our Mongo instance

In this section you will create an Ubuntu Linux instance using an `OS Only` blueprint. As part of the instance creation process you'll supply a launch script that will install MongoDB. 

1) From the Lightsail console home page click `Create Instance`

    ![](./images/2-1-1.jpg) 

2) Under `Blueprint` click `OS Only` and choose `Ubuntu`

    ![](./images/2-1-2.jpg) 

3) Click `+Add Launch Script`

    ![](./images/2-1-3.jpg) 

4) Paste the script below into the text box

        #!/bin/bash
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4
        echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.0.list
        sudo apt-get update
        sudo apt-get install -y mongodb-org

        sudo cat /etc/mongod.conf | sed "s/\b127.0.0.1\b/&,$(hostname -i)/" >> /etc/mongod.conf.new
        
        sudo mv /etc/mongod.conf /etc/mongd.conf.old
        sudo mv /etc/mongod.conf.new /etc/mongod.conf

        sudo service mongod start

    This script does the following:

    * Uses `apt-get` to install Mongo
    * Modifies the Mongo configuration file to make sure Mongo is listening on the instances private IP address
    * Backs up the old configuration file and copies in the new one
    * Starts Mongo

5) Scroll down and enter `Mongo` as the Instance name

    ![](./images/2-1-5.jpg)

6) Click `Create`

    ![](./images/2-1-6.jpg)

7) Once the instance is up and running from the Lightsail home page click on the instance name. This will bring up the instance details page

    ![](./images/2-1-7.jpg)

8) Document the private IP of the instance. 

    ![](./images/2-1-8.jpg)

    ***Note:** You will use this IP in the next section*

###2.2 - Create the web front end instance
Next you are going to create our web front end instance. The front end instance will use the Lightsail NodeJS blueprint along with the application code. Additionally it will use a process manager, PM2, to ensure that our application starts up when the instance boots. 

Check out the [PM2 website](http://pm2.keymetrics.io/) to learn more about this tool. 

1) From the Lightsail console click `Create`

2) Under `Blueprint` choose `NodeJS`

3) Click `+Add Launch Script` and paste in the script below

        #!/bin/bash

        sudo /opt/bitnami/ctlscript.sh stop apache
        sudo mv /opt/bitnami/apache2/scripts/ctl.sh /opt/bitnami/apache2/scripts/ctl.sh.disabled

        cd /home/bitnami

        sudo git clone https://github.com/mikegcoleman/todo

        cd /home/bitnami/todo

        sudo npm install --production
        sudo npm install pm2@latest -g

    This script:

    * Disables Apache to free up port 80 for the web front end
    * Clones in the application code from the GitHub repo
    * Uses `npm` to install the application dependencies and PM2

4) Name the instance `node-fe-1`

5) Click `Create`

Wait for the instance to reach a running state, and then SSH into the instance

***Note:** The following commands need to be run from the command line of the NodeJS isntance

6) Set an environment variable to hold the MongoDB private IP address by executing the command below and substituting the IP address your recorded above.

        IP = <MongoDB instance private IP>

    For example:

        IP = 172.12.14.1
7) Create a config file to hold our environment variables by pasting the following lines at the command prompt in the NodeJS instance. 

        sudo cat <<EOF >> ~/todo/.env
        PORT=80
        DB_URL=mongodb://$(echo IP):27017/
        EOF

These variables specify the port the application will run on, and the connection string for the MongoDB host

8) Configure PM2 for use with Ubuntu by issuing the following command
        
        sudo pm2 startup ubuntu

9) Start the application using PM2

        sudo pm2 start /home/bitnami/todo/bin/www

10) Save out the current process list for PM2. This will allow PM2 to start the application when the instance boots up subsequently
        sudo pm2 save