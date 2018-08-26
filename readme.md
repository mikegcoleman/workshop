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

        sudo cat << EOF >> /home/bitnami/todo/.env
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

    You should get a message that you successfully added the user:

        Successfully added user: { "user" : "tasks", "roles" : [ "dbOwner" ] }

4) Close the mongo shell by typing exit

        exit

### 1.3 - Start and test our application

Now that the application is installed and the database configured it is ready to be started. 

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

To fix this issue the front end and database each need to be running on their own instance. In this lab you will create an instance for the database and one for the web front end. Then you will take a snapshot of the web front end, and deploy two additional web front end instances from that snapshot. Finally, you'll add a loadbalancer in front of the three front end instances. 

### 2.1 - Create our Mongo instance

In this section you will create an Ubuntu Linux instance using an `OS Only` blueprint. As part of the instance creation process you'll supply a launch script that will install MongoDB. 

1) From the Lightsail console home page click `Create Instance`

    ![](./images/2-1-1.jpg) 

2) Under `Select a blueprint` click `OS Only` and choose `Ubuntu`

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

7) Once the instance is up and running SSH into the box (username: `ubuntu`)

8) Use the Mongo client to ensure that the machine is running correction

        mongo --host $(hostname -i)

9) From the Mongo client to show the built in databases

        show dbs

    The output should be:

        admin   0.000GB
        config  0.000GB
        local   0.000GB

10) Type `exit` to close the mongo client

11) In your web browser return to the Lightsail console

12) From the Lightsail home page click on the instance name. This will bring up the instance details page

    ![](./images/2-1-7.jpg)

8) Document the private IP of the instance. 

    ![](./images/2-1-8.jpg)

    ***Note:** You will use this IP in the next section*

### 2.2 - Create the web front end instance
Next you are going to create our web front end instance. The front end instance will use the Lightsail Node.js blueprint along with the application code. Additionally it will use a process manager, PM2, to ensure that our application starts up when the instance boots. 

Check out the [PM2 website](http://pm2.keymetrics.io/) to learn more about this tool. 

1) From the Lightsail home page click `Create Instance`

    ![](./images/2-2-1.jpg)

2) Under `Select a blueprint` choose `Node.js`

    ![](./images/2-2-2.jpg)

3) Click `+Add launch script`

     ![](./images/2-2-3.jpg)

4) Paste in the script below

        #!/bin/bash

        sudo /opt/bitnami/ctlscript.sh stop apache
        sudo mv /opt/bitnami/apache2/scripts/ctl.sh /opt/bitnami/apache2/scripts/ctl.sh.disabled

        cd /home/bitnami

        sudo git clone https://github.com/mikegcoleman/todo

        cd /home/bitnami/todo

        sudo npm install --production
        sudo npm install pm2@latest -g
      

    This script does the following:

    * Disables Apache to free up port 80 for the web front end
    * Clones in the application code from the GitHub repo
    * Uses `npm` to install the application dependencies and PM2
    

5) Name the instance `node-fe-1`

    ![](./images/2-2-5.jpg)

6) Click `Create`

    ![](./images/2-2-6.jpg)

7) Wait for the instance to reach a running state, and then SSH into the instance (username: `bitnami`)

    ***Note:** The following commands need to be run from the command line of the NodeJS isntance*

6) Set an environment variable to hold the MongoDB private IP address by executing the command below and substituting the IP address your recorded above.

        IP=<MongoDB instance private IP>

    For example:

        IP=172.12.14.1

7) Ensure the IP is correctly set by typing the command below. The output should be the private IP of the MongoDB instance

        echo $IP

8) Change into the application directory

        cd ~/todo

7) Create a config file to hold our environment variables by pasting the following lines at the command prompt in the NodeJS instance. 

```
sudo sh -c "cat > ./.env"  << EOF
PORT=80
DB_URL=mongodb://$IP:27017/
EOF
```

These variables specify the port the application will run on, and the connection string for the MongoDB host

8) Ensure the file was created succesfully by using `cat` to list out the contents

        cat ./.env

    The output should be similar to below (ensure the IP address matches the **PRIVATE IP** of your Mongo instance)

        PORT=80
        DB_URL=mongodb://IP=172.12.14.1:27017/
        
8) Configure PM2 for use with Ubuntu by issuing the following command
        
        sudo pm2 startup ubuntu

9) Start the application using PM2

        sudo pm2 start /home/bitnami/todo/bin/www

10) Save out the current process list for PM2. This will allow PM2 to start the application when the instance boots up subsequently

        sudo pm2 save

11) Issue the following command to have PM2 stream the application logs to the console

        sudo pm2 logs www

11) Ensure everything is working by navigating the the **PUBLIC IP** of your `MEAN` Instance

    You should see the todo application in your web browser and if you check the SSH session there should be some output there as well similar to the below:

        0|www  | GET / 200 1005 - 42.341 ms
        0|www  | GET /stylesheets/style.css 200 89 - 2.430 ms
        0|www  | GET /css/bootstrap.min.css 200 140936 - 3.865 ms
        0|www  | GET /css/fontawesome-all.css 200 51609 - 0.807 ms
        0|www  | GET /js/bootstrap.min.js 200 51039 - 4.965 ms
        0|www  | GET /js/jquery.min.js 200 86927 - 2.875 ms

    ***Note:** You can find the public IP on the card with the instance name for the `MEAN` instance on the Lightsail home page.* 

    ![](./images/2-2-13.jpg)


## 2.3 Creating additional web front-end instances

1) Return to the Lightsail console home page

2) Click the 3 dot menu for the `node-fe-1` instance and choose `Manage`

    ![](./images/2-3-2.jpg)

3) From the horizontal menu choose `Snapshots`

    ![](./images/2-3-3.jpg)

4) Click Create Snapshot

    ![](./images/2-3-4.jpg)

    Under `Recent snapshots` the status will change to `creating`, you will need to wait for the process to complete to move forward. This can take up to 5 minutes. 

5) Click the 3 dot menu to the right of the newly created snapshot and select `Create new instance`

    ![](./images/2-3-5.jpg)

6) Scroll down and name the instance `node-fe-2`

7) Click `Create`

8) Click the 3 dot menu for the `node-fe-1` instance and choose `manage`

    ![](./images/2-3-2.jpg)

9) From the horizontal menu choose `Snapshots`

    ![](./images/2-3-3.jpg)

10) Click the 3 dot menu to the right of the previously created snapshot and select `Create new instance`

    ![](./images/2-3-5.jpg)

11) Scroll down and name the instance `node-fe-3`

12) Click `Create`

13) Test the public IP of each of the two newly created front end instances in your web browser. Notice that the hostname for that particular web front end instance is listed under your task list, and that it changes based on which instance you are visiting in your web browser

    ![](./images/2-3-13.jpg)

### 2.4 - Load balance the web front-end

1) Return the the Lightsail home page and choose `Networking` from the horizontal menu. 

    ![](./images/2-4-1.jpg)

2) Click `Create Load Balancer`

    ![](./images/2-4-2.jpg)

3) Scroll down and enter `todo-lb` for the load balancer name

    ![](./images/2-4-3.jpg)

4) Click `Create Load Balancer`

    ![](./images/2-4-4.jpg)

5) Under `Target instances` choose `node-fe-1` from the dropdown list

    ![](./images/2-4-5.jpg)

6) Click `atttach`

    ![](./images/2-4-6.jpg)

7) Click `Attach another` and repeat steps 5 and 6 for `node-fe-2` and `node-fe-3`

    ![](./images/2-4-7.jpg)

    ***Note:** It will take several minutes for all three instances to register their health checks as `Passed` once this has happened, move to the next step*
    
    ![](./images/2-4-7a.jpg)

8) From the top of the screen copy the long string following `DNS name:`. This is the URL for your Lightsail load balancer. Any requests on this URL will be routed to one of your three front end instances. 

    ![](./images/2-4-8.jpg)

9) Paste the string into a web browser, the Todo application should come up. Reload the page and notice how the host name at the bottom of the screen changes - this indicates that traffic is being routed appropriately. 

## Cleanup
Be sure to either stop or delete all the resources from this lab so as not to incur additional charges. To cleanup the resources follow the steps below. 

1) Navigate back to the Lightsail home page

2) Delete the 5 lightsail instances you created by clicking the 3 dot menu on each instance and choosing `delete`. You will need to confirm each action by clicking `Yes, delete`

3) Click `Networking` from the horizontal menu

4) Click the 3 dot menu on the `todo-lb` load balancer and choose `Delete`. You will need to confirm the action by clicking `Yes, delete`



