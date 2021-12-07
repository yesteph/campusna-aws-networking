# campusna-aws-networking

# Customize an EC2 instance with user-data and create an AMI

In this exercise, you will start an EC2 instance and configure it with `user-data` to install an Apache server and format and mount an EBS volume.

This EBS volume will contains the web server content - directory `/var/www/html`.

Then, you will connect the instance to change the web content.
At this stage, you can create an AMI to be able to persist the current configuration and launch several other instances.

Finally, you create a new instance with the created AMI.