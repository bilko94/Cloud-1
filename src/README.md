# cloud-1
### Score 100/100
#### Mandatory
100/100

#### Bonus
N/A

WeThinkCode_ DevOps module - Project 1

Summary: Introduction to databases, data and cloud servers

---
## Project Requirements:
Install a simple wordpress site on cloud infrastructure.

* Website must be running on at least 2 servers (if possible, in different server farms).
* A Load Balancer to redirect requests to any of the available servers.
* Traffic spikes will cause new servers to launch, and traffic decreases will cause new servers to shutdown.
* Logged in users will stay identified for the length of a normal session.
* New content on the site should be available across all instances as soon as possible.
* Static content should be distributed via CDN.
* Failures should be handled so that website uptime is not interrupted.
* Hosting cost should reflect actual usage.
* Appropriate firewall rules / protection measures should be in place as to avoid inappropriate access.

---
## Google Cloud Platform
$300 free credit was used.

This project was setup as follows:

1. Register an account on GCP (may be eligible for $300 free credit)
2. Create a new project.
3. Use the following [Click to Deploy package](https://console.cloud.google.com/marketplace/details/click-to-deploy-images/wordpress?q=wordpress) to start a basic Wordpress webserver VM. Configure it to your liking. Recommend using a region closest to you.
4. Create a Cloud Storage bucket, named as you choose, and then...
    - Go to the 'IAM & Admin > Service Accounts' page and create a new Service Account.
    - Assign the Service Account the role of `Cloud SQL > Cloud SQL Client` & `Project > Editor`.
    - Download the key in JSON format and keep it handy.
5. Go to VPC > External IP Addresses
    - Reserve a Static IP, name it `Loadbalancer static` and ensure it is Premium tier, IPv4, and type is Global.
    - If you're using a domain, you will need to update your DNS A Record to point to this IP address.
6. Configure Wordpress & the server
    1. SSH to the VM
        - `sudo su`
        - `cd /var/www/html`
        - Create an `.htaccess` file, add the following (Note the commented out entries for later):
            ```
            #RewriteEngine On
            #RewriteCond %{HTTP:X-Forwarded-Proto} =http
            #RewriteRule .* https://%{HTTP:Host}%{REQUEST_URI} [L,R=permanent]
            ```
        - Add the following to your `wp-config.php` file, below the Database definitions (Note the commented out entries for later):
            ```
            define( 'AS3CF_SETTINGS', serialize( array(
                'provider' => 'gcp',
                    'key-file-path' => '/etc/keyFile.json',
            ) ) );

            #define('FORCE_SSL_ADMIN', true);
            #if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false)
            #    $_SERVER['HTTPS']='on';

            ```
        - `cd /etc/ && vim keyFile.json`
        - Add the contents of the previously downloaded JSON file to `keyFile.json`
        - `cd /var/www/html/wp-content/themes/twentyseventeen/template-parts/footer`
        - `vim site-info.php`
        - Adjust the line that contains the 'printf Proudly powered by' text to look as follows:

            `printf( __( 'Proudly powered by %s<br>%s', 'twentyseventeen' ), 'WordPress', $_SERVER['SERVER_ADDR'] );`
        
            This displays the current servers internal IP in the footer, to ensure the loadbalancer is working.

    2. Login to phpMyAdmin via `ExternalIP/phpmyadmin`, using the details contained within the VM info page.
        - Open the `Wordpress` database.
        - Export the database (SQL format, Custom Export - All tables, charset: `utf8mb4`, collate: `utf8mb4_general_ci`)
        - Download the exported SQL, and upload it to your bucket.
        - Create a new Cloud SQL Instance (MySQL):
            1. Under the Configuration Options drop down: The machine type I used was `db-n1-standard-1`, SSD Disk, 25Gb in size.
            2. Under connectivity, enable Private IP.
            3. Ensure High Availability is selected under the Backups... tab
        - Create a new database, named as you please, with charset and collate settings matching the exported ones above.
        - Import the SQL backup you saved to your bucket and select your new database in the drop down.
        - Create a new user, allow them access to anyhost. Keep the username and password handy.
        - Once again, edit the `wp-config.php` file, and change the following to their relevant details:
            ```
            define( 'DB_NAME', 'Your SQL Database Name' );
            define( 'DB_USER', 'Your SQL Database Username' );
            define( 'DB_PASSWORD', 'Your SQL Database User Password' );
            define( 'DB_HOST', 'Private IP Address of SQL Instance' );
            define( 'DB_CHARSET', 'utf8mb4' );
            define( 'DB_COLLATE', 'utf8mb4_general_ci' );
            ```
    3. Login to wordpress admin via `ExternalIP/wp-admin`, using the details contained within the VM info page.
        - Install & Activate the `WP Offload Media` & `W3 Total Cache` plugins.
        - In the `WP Offload Media` settings:
            - Select Google Cloud Storage > Define Key File (already done)
            - Press next & enter the name of your previously created Storage Bucket.
        - In the `W3 Total Cache` settings:
            - In the CDN sub section of General Settings, Enable CDN and use Generic Mirror
            - Back at the top, Click on Show Required Changes and add the text displayed to your `.htaccess` file, below the commented out lines already there.
        - ONLY ONCE YOU ARE SATISFIED WITH THE WORDPRESS CONFIGURATION:
            - Change the 2 site URL fields in the General Wordpress settings to either point to your domain (if you're using one) or the load balancer IP.
    4. Finally, you can uncomment the 3 lines in each of the `.htaccess` and `wp-config.php` files that deal with HTTPS.

7. Under the Compute Engine tab:
    - Shut down the running instance.
    - Go to the `Images` tab, create a new custom image using the Disk of the VM as source.
    - Go to the `Instance Template` tab, create a new template, set the machine type to your liking, change the boot disk to a custom image type and use the one you just created, and allow HTTP / HTTPS traffic.

8. Still under the Compute Engine tab:
    - Go to the `Instance Group` tab and create a new group. 
        - Use the template you just created. 
        - Ensure autoscaling is ON, use Multiple Zone location (in the same region as your SQL instance). 
        - Minimum instances = 2, maximum instances = 6. 
        - Create a health check on TCP port 80.

9. VPC > Firewall:
    - Create a new rule:
        - Ensure Direction is Ingress
        - Targets = All Instances
        - Source Filter = IP Ranges
        - Ranges = `130.211.0.0/22, 35.191.0.0/16`
        - Specific TCP port of 80.

## If using a domain:
    10. Network Services > Loadbalancing:
        1. HTTPS Loadbalancer:
            - Create a new HTTP/S Load balancer
            - From Internet to VMs
            - Create a new backend service:
                - HTTP protocol over port 80
                - Instance group = your instance group
                - Enable Cloud CDN
                - Apply your HTTP health check.
            - Create a new frontend service:
                - HTTPS protocol over port 443
                - Set the IP address as your previously reserved static IP address.
                - Create a new SSL certificate (Google Managed, use your domain name or Loadbalancer IP address [Not Tested myself with loadbalancer ip])
            - Finish
        2. HTTP Loadbalancer:
            - Create a new HTTP/S Load balancer
            - From Internet to VMs
            - Skip Backend Service
            - Host Path and Rules:
                - Advanced
                - Redirect to Different Host/Path
                - Prefix Redirect
                - Status 301
                - Enable HTTPS redirect
            - Create a new frontend service:
                - HTTP protocol over port 80
                - IP address is previously reserved static IP address.


## If NOT using a domain:
    10. Network Services > Loadbalancing:
        1. HTTP(S) Loadbalancer:
            - Create a new HTTP/S Load balancer
            - From Internet to VMs
            - Create a new backend service:
                - HTTP protocol over port 80
                - Instance group = your instance group
                - Enable Cloud CDN
                - Apply your HTTP health check.
            - Create a new frontend service:
                - HTTP protocol over port 80
                - IP address is previously reserved static IP address.

---

11. You may have to wait a little while for everything to spin up and start talking to one another, but accessing your domain / loadbalancer IP address should get you to the Wordpress homepage.



---
# Notes

- I'm not certain whether using the loadbalancer IP for the SSL cert will work.
- You may be able to find a free domain somewhere, I used one I already owned.


## Resources used:
- Various Google Cloud Docs
- https://geekflare.com/google-cloud-cdn-test/
- https://geekflare.com/gcp-load-balancer/
- https://themeisle.com/blog/install-wordpress-on-google-cloud/
- https://www.cloudbooklet.com/best-performance-wordpress-with-google-cloud-cdn-and-load-balancing/
- https://www.wpexplorer.com/install-wordpress-google-cloud/
- http://www.mindinfinity.net/autoscaling-with-google-cloud/index.html