# Heralding-Frontend
###Disclaimer: 
A certain amount of knowlege and technical expertise is expected in networking, linux, and mysql, and as such support in those areas on my part will be limited. The project goal is to get the Frontend out to the public to try out. The instructions below include example configurations and should be used at your own risk.

###About Heralding-Frontend:
For a very long time now, I had this desire to set up a honeypot and use it to create a blacklist that my firewall could use to block potential attackers. After digging around a bit I found Heralding, and then later a very useful tutorial by Dennis Puerto that got me going in the right direction. After initially building a completely text based version of the gui (loosely based on Dennis's project here: https://www.linkedin.com/pulse/honeypot-generating-blacklists-cisco-firepower-dennis-perto/), I figured I could take it up a notch by creating something that scales much better with larger datasets that could be used for years at a time. 

Heralding-Frontend is a database driven self contained Dotnet Core web frontend for a stock install of the Heralding honeypot. The intent of this application is to provide a free searchable GUI for reporting Heralding traffic by moving the honeypot connection data to a database for processing. Using MySQL as a free backend data storage platform, new features become available like creating blacklists for firewall ingestion, password lists, or generate username lists to compare against your AD users. 

###Environment:
The honeypot & frontend seem to run perfectly well on a guest vm with 2gb of ram with 2 processors and 20Gb of storage space all on an optiplex 7010 (host: Windows 10 20h2 i7 3.4ghz, 8gb ram, 250gb ssd from 2014) running Ubuntu 20.04 fully patched. The honeypot also has a single NIC with multiple public IP's which provides for a larger attack surface with more flexibility. While the system is quite responsive, you may need to add more ram depending on your setup and so I recommend 4gb with 4 processors and an ssd. In the following setup instructions, the frontend application resides in the Heralding directory in a folder called "frontend" for simplicity. On my dev lab, I have Heralding installed to the users home directory, but can be physically exist anywhere as long as the scheduled tasks, services, and scripts all point to the right location. 

###Installing Heralding-Frontend:
The following instructions, generally speaking, walk through installation of Dotnet Core, MySQL, configuring services, and scheduled jobs in 7 or 8 easy steps. In addition to creating the service for the frontend, it works best if Heralding itself also starts as a service. 

1. This application assumes Heralding is already installed and collecting logs into the session & auth .csv's. If not, please visit: https://github.com/johnnykv/heralding/blob/master/INSTALL.md
  **Note: Don't forget to configure the firewall (use what ever firewall you are confortable with) for any ports you want to monitor. In addition, you'll need to open a port for internal private access to the website, like 8181 in the example below. Do not open this port to the outside world!!!!**
  
2. At this point you can move the application in to {Heralding install directory}/frontend and the .sh scripts to {heralding install directory}/frontend-services/.**
     
3. Dotnet Core 3.1 (support for 5.0 will be made available as soon as Pomelo comes out of Alpha). To install, please visit: https://docs.microsoft.com/en-us/dotnet/core/install/linux-ubuntu
   -Test to make sure the honeypot is available at http://127.0.0.1:5000 before moving on.
   
4. Install and configure Apache2 reverse proxy: https://www.c-sharpcorner.com/article/how-to-deploy-net-core-application-on-linux/
  **Note: The port apache listens on cannot be one already in use or monitored by heralding. Make sure to change the listening port and add a .conf file to your conf-enabled folder. In the following example, Apache is configured to listen on port 8181 and forward traffic to the internal Kestrel server that DotNet uses.**

   /etc/apache2/conf-enabled/heralding-frontend.conf:
 >     <VirtualHost *:8181>  
 >      ProxyPreserveHost On
 >      ProxyPass / http://127.0.0.1:5000/
 >      ProxyPassReverse / http://127.0.0.1:5000/
 >      ErrorLog ${APACHE_LOG_DIR}/heralding-frontend-error.log  
 >      CustomLog ${APACHE_LOG_DIR}/heralding-frontend-access.log combined  
 >     </VirtualHost>
      
   /etc/pache2/ports.conf:
>      ...
>      Listen 8181
>      ...
      
   /etc/systemd/system/heralding-frontend.service:
>      [Unit]
>      Description=Heralding Frontend (DotNet 3.1)
>      [Service]
>      WorkingDirectory=#{Heralding install directory}/frontend
>      ExecStart=/usr/bin/dotnet #{Heralding install directory}/frontend/HeraldingFrontend.dll
>      Restart=always
>      RestartSec=10
>      SyslogIdentifier=netcore-demo
>      User=www-data
>      Environment=ASPNETCORE_ENVIRONMENT=Production
>      [Install]
>      WantedBy=multi-user.target
      
>      sudo systemctl enable heralding-frontend
>      sudo systemctl start heralding-frontend
   
5. This application uses MySQL 8.0.21 as a backend to store Heralding logs. An easy tutorial to follow for installing is here: https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04

  **Note: Please remember to change the MySQL port if Heralding is configured to monitor for MySQL logins on port 3307. 
          In the following example, MySQL is configured to use port 3307.**
          
   mysqld.cnf:
>      [mysqld]
>      ...
>      port		= 3307 #(or anything you want that isn't already in use or being monitored)
>      local_infile	= 1 # Required for moving data into MySQL
>      bind-address		= 0.0.0.0
>      secure_file_priv	= ''
>      general_log_file	= /var/log/mysql/mysql-General.log
>      general_log		= 1

6. 2 jobs should be configured to run the two scripts to both move data in to sql and also move exported MySQL data to the public website. The following example runs those processes every 5 minutes but they can be changed as necessary.

>   sudo crontab -e:
>      */5 * * * * sh {heralding install directory}/frontend-services/mv_to_mysql.sh >> /var/log/heralding-mysql.log
>      */5 * * * * sh {heralding install directory}/frontend-services/update.sh >> /var/log/heralding-update.log
    
   **Note: mv_to_mysql.sh uses the encrypted login-path called "mypath" instead of hard coding the login information: https://dev.mysql.com/doc/refman/5.6/en/option-file-options.html**
   **Important Note: Each script is configured using variables at the top of each script indicating where the directory Heralding is installed. It must be changed to match your configuration.**

7. Copy the Heralding Frontend code to {heralding install directory}/frontend and in the appsettings.json file, change the HoneypotDBConnection port number, username, and password. Now you are ready to test the site.

8. Optionally, create scheduled tasks in MySQL to export data as needed so the heralding-update service (in step 5) cam move the files out to the public folder. If workbench is installed, just copy and paste the following and execute:

> CREATE EVENT make_30_day_blacklist ON SCHEDULE EVERY 5 MINUTE STARTS CURRENT_TIMESTAMP DO Select distinct source_ip from sessions left join whitelist on source_ip = ip_addr where ip_addr is null and timestamp between subdate(curdate(), 30)and curdate()  INTO OUTFILE '/var/lib/mysql-files/30-day-blacklist.txt' LINES TERMINATED BY '\r\n';

> CREATE EVENT make_14_day_blacklist ON SCHEDULE EVERY 5 MINUTE STARTS CURRENT_TIMESTAMP DO Select distinct source_ip from sessions left join whitelist on source_ip = ip_addr where ip_addr is null and timestamp between subdate(curdate(), 14)and curdate()  INTO OUTFILE '/var/lib/mysql-files/14-day-blacklist.txt' LINES TERMINATED BY '\r\n';

> CREATE EVENT make_blacklist ON SCHEDULE EVERY 5 MINUTE STARTS CURRENT_TIMESTAMP DO select distinct source_IP from sessions left join whitelist on source_ip = ip_addr where ip_addr is null INTO OUTFILE '/var/lib/mysql-files/blacklist.txt' LINES TERMINATED BY '\r\n';

> CREATE EVENT make_whitelist ON SCHEDULE EVERY 5 MINUTE STARTS CURRENT_TIMESTAMP DO Select distinct ip_addr from whitelist INTO OUTFILE '/var/lib/mysql-files/whitelist.txt' LINES TERMINATED BY '\r\n';

> CREATE EVENT make_password_list ON SCHEDULE EVERY 5 MINUTE STARTS CURRENT_TIMESTAMP DO Select distinct password from authentications order by password INTO OUTFILE '/var/lib/mysql-files/passwordlist.txt' LINES TERMINATED BY '\r\n';

> CREATE EVENT make_username_list ON SCHEDULE EVERY 5 MINUTE STARTS CURRENT_TIMESTAMP DO Select distinct username from authentications order by username INTO OUTFILE '/var/lib/mysql-files/usernamelist.txt' LINES TERMINATED BY '\r\n';

> CREATE EVENT make_14_day_username_list ON SCHEDULE EVERY 5 MINUTE STARTS CURRENT_TIMESTAMP DO Select distinct username from authentications where timestamp between subdate(curdate(), 14) and curdate() order by username INTO OUTFILE '/var/lib/mysql-files/14-day-username-list.txt' LINES TERMINATED BY '\r\n';
    
