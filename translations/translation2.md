# LAB: Google Cloud Fundamentals: Getting Started with Cloud Storage and Cloud SQL
## Objectives: 


In this lab, you learn how to perform the following tasks:

 1.   Create a Cloud Storage bucket and place an image into it.
 2.   Create a Cloud SQL instance and configure it.
 3.   Connect to the Cloud SQL instance from a web server.    
 4.   Use the image in the Cloud Storage bucket on a web page.
    
## Steps:
 #### Deploy a web server VM instance
 1. In your shell, run the following command to create a VM instance called **bloghost**. All unspecified flags will remain at their default values.

	```
	gcloud compute instances create bloghost \
	--zone=us-central1-a --machine-type=e2-medium --subnet=default \
	--metadata=startup-script=apt-get\ update$'\n'apt-get\ install\ apache2\ php\ php-mysql\ -y$'\n'service\ apache2\ \
	restart --tags=http-server --image=debian-9-stretch-v20200910 \
	--image-project=debian-cloud --boot-disk-size=10GB \
	--boot-disk-type=pd-standard --boot-disk-device-name=bloghost
	```
2.  Now we need to create an INGRESS firewall rule to allow HTTP traffic
	```
	gcloud compute firewall-rules create default-allow-http \
	--direction=INGRESS --priority=1000 --network=default \
	--action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 \
	--target-tags=http-server
	```
3. Copy the **Internal** address for the the bloghost instance which can be obtained by the following command and save it in a text editor to be reused later
	```
	gcloud compute instances describe bloghost \
	  --format='get(networkInterfaces[0].networkIP)'
	```
4. Copy the **External** address for the the bloghost instance which can be obtained by the following command and save it in a text editor to be reused later
	```
	gcloud compute instances describe bloghost \
	  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
	```
 #### Create a Cloud Storage bucket using the gsutil command line
 1. Enter your chosen location into a environment variable called LOCATION
 ```export LOCATION=US```
 or
 ```export LOCATION=EU```
 or
 ```export LOCATION=ASIA```
 2. Now we will make use of the DEVSHELL_PROJECT_ID environment variable which automatically contains our Project ID, and specify the location for the bucket to be created.
```gsutil mb -l $LOCATION gs://$DEVSHELL_PROJECT_ID```
 3. Retrieve a banner image from a publicly accessible Cloud Storage location:
 ```gsutil cp gs://cloud-training/gcpfci/my-excellent-blog.png my-excellent-blog.png```
 4. Copy the banner image to your newly created Cloud Storage bucket:
  ```gsutil cp gs://cloud-training/gcpfci/my-excellent-blog.png my-excellent-blog.png```
  5. Modify the Access Control List of the object you just created so that it is readable by everyone:
  ```gsutil acl ch -u allUsers:R gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png```
  
#### Create the Cloud SQL instance
1. Use  the gcloud sql command given below to create a SQL instance with  instance ID of blog-db and a root password
``` gcloud sql instances create blog-db --database-version=MYSQL_5_7 --tier=db-n1-standard-1 --region=us-central1 --root-password=pass1324```
2. Copy the public IP address given when you use the following describe command for the SQL instance for later use
```gcloud sql instances describe blog-db```
3. Create a new user called bloguserdb
```gcloud sql users create bloguserdb --instance=blog-db --password=12345678```
4. Add a network connection using the following commands. In the second step, we will use the Public IP of our bloghost VM instance and add /32 (as an example, I will assume the IP is 35.192.208.2)
	```
	gcloud sql instances patch blog-db --assign-ip 
	gcloud sql instances patch blog-db --authorized-networks=35.192.208.2/32
	gcloud sql instances describe blog-db
	```
#### Configure an application in a Compute Engine instance to use Cloud SQL
1. SSH into the VM instance bloghost
	```
	gcloud compute ssh bloghost --zone=us-central1-a
	```
2. In your ssh session on **bloghost**, change your working directory to the document root of the web server:
```cd /var/www/html```
3. Use the **nano** text editor to edit a file called **index.php**
```sudo nano index.php```
4. Paste the following content into the file:
	```
	<html>
	<head><title>Welcome to my excellent blog</title></head>
	<body> 
	<h1>Welcome to my excellent blog</h1> 
	<?php 
	$dbserver = "CLOUDSQLIP"; 
	$dbuser = "blogdbuser"; 
	$dbpassword = "DBPASSWORD"; 
	// In a production blog, we would not store the MySQL 
	// password in the document root. Instead, we would store it in a 
	// configuration file elsewhere on the web server VM instance. 
	
	$conn = new mysqli($dbserver, $dbuser, $dbpassword);

	if (mysqli_connect_error()) { 
		echo ("Database connection failed: " . mysqli_connect_error()); 
	} else { 
		echo ("Database connection succeeded."); 
	} 
	?> 
	</body></html>
	```
5. Press **Ctrl+O**, and then press **Enter** to save your edited file.
6.  Press  **Ctrl+X**  to exit the nano text editor.
7. Restart the web server:
```sudo service apache2 restart```
8. Open a new web browser tab and paste into the address bar your **bloghost** VM instance's external IP address followed by **/index.php**. The URL will look like this (make sure you use your VM external IP address):
```35.192.208.2/index.php```
When you load the page, you should see the words `Database connection failed: ...`
9.  Return to your ssh session on  **bloghost**. Use the  **nano**  text editor to edit  **index.php**  again.
	```
	sudo nano index.php
	```
10.  In the  **nano**  text editor, replace  `CLOUDSQLIP`  with the Cloud SQL instance Public IP address that you noted above. Leave the quotation marks around the value in place.
11.  In the  **nano**  text editor, replace  `DBPASSWORD`  with the Cloud SQL database password that you defined above. Leave the quotation marks around the value in place.
12.  Press  **Ctrl+O**, and then press  **Enter**  to save your edited file.
13.  Press  **Ctrl+X**  to exit the nano text editor.
14.  Restart the web server:
		```
		sudo service apache2 restart
		```
15.  Return to the web browser tab in which you opened your  **bloghost**  VM instance's external IP address. When you load the page, the following message appears:
		```
		Database connection succeeded.
		```