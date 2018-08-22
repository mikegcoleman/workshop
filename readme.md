## Demo #1 - Deploying a Standalone MEAN Application
To start we're going to deploy a MEAN stack application into a single Lightsail instance. This means that both the Express front end and the Mongo database will be running on the same host. We'll build a new application instance, configure the Mongo database, and then test out our application 

### 1.1: Build the MEAN application instance
The first thing we need to do is create a MEAN stack instance in Lightsail. When we create our instance, we'll use a Lightsail launch script to install our application.

1) From the Lightsail console click 'Create Instance'

2) Under 'Blueprint' choose 'MEAN'

3) Click '+Add Launch Script'

4) Paste the script below into the window

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

    * Stops Apache using the control scripts present in the Bitnami MEAN image. Stopping Apache frees up Port 80 so our application can use it. 
    * Clones the application GitHub repo and installs all the dependencies using the Node Package Manager (`npm`)
    * Creates a configuration file that sets the application port (`80`) and the connection string for the database running on the loca lhost (`mongodb://tasks:tasks@localhost:27017/?authMechanism=SCRAM-SHA-1&authSource=tasks`)

5) Once the instance shows a state of running in the Lightsail console, go ahead and SSH into it either using the built in SSH client or using your own (username: `bitnami`). If you're unfamiliar with SSH please see this tutorial. 

    ***Note**: Even though the instance shows a state of running, it may still be executing our startup script, and you won't be able to connect. If this is the case, give it a couple of minutes and try again.*
    
### 1.2 - Configure the Mongo database

In this section we're goin to configure the Mongo database for our application. Specifically we're going to add a username and password  (`tasks` for both) to our Mongo database (also called `tasks`). We'll do all this using the Mongo client from the Lightsail command line. 

***Note**: The following steps are performed from the Lightsail instance command line*

1) First log into the mongo client using the following command

    ***Note**: Each Bitnami-based Lightsail instance stores the application password in a file called `bitnami_application_password`. Below we're redirecting that file into the Mongo client command line.*

        mongo admin --username root -p $(cat ./bitnami_application_password)

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

4) Type 'exit' to close the mongo shell

### 1.3 - Start and test our application

Now that we've installed the application and configured the database, we can now run the application. 

1) Change into the application directory

        cd ~/todo

2) Start the application by executing the following command
    
        sudo DEBUG=app:* ./bin/www

3) From the Lightsail console get the IP address of your MEAN instance and navigate to that address in your web browser. 

You should see the ToDo application running. Add a task or two to make sure it's working as expected. 

## Demo #2 - Scaling the Todo Application

### 2.1 - Create our Mongo instance

Begin by creating a Lightsail instance and installing Mongo. 

1) From the Lightsail console click 'Create Instance'
 
2) Under 'Blueprint' select 'OS Only' and choose 'Ubuntu'

4) Click '+Add Launch Script' and paste in the script below

        #!/bin/bash
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4
        echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.0.list
        sudo apt-get update
        sudo apt-get install -y mongodb-org

        sudo cat /etc/mongod.conf | sed "s/\b127.0.0.1\b/&,$(hostname -i)/" >> /etc/mongod.conf.new
        
        sudo mv /etc/mongod.conf /etc/mongd.conf.old
        sudo mv /etc/mongod.conf.new /etc/mongod.conf

        sudo service mongod start

    This script:

    * Uses `apt-get` to install Mongo
    * Modifies the Mongo configuration file to make sure Mongo is listening on the instances private IP address
    * Backs up the old configuration file and copies in the new one
    * Starts Mongo

5) Scroll down and enter 'Mongo' as the Instance name

6) Click 'Create'

7) Once the instance is up and running you need to document the private IP of the instance. You can find this by clicking on the the instance name. You will use this IP in the next section.

###2.2 - Create the web front end instance
Next we're going to create our web front end instance. We'll use the NodeJS blueprint and add in the application code. Additionally we'll use a process manager, PM2, to ensure that our application starts up when the instance boots. 

Check out the PM2 website to learn more about this tool. 

1) From the Lightsail console click 'Create'

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