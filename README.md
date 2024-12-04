## Prerequisites 
- A clone of the provided Git repository 
- nginx and ufw packages installed

<br>

## Create Your Droplets and Load Balancer

1. Create your 2 droplets on digital ocean with the tag "web"

* Refer to digitalOcean documentation if you need help setting these up

<br>

2. Create a load balancer for the droplets

* Refer to digitalOcean documentation if you need help

**Load Balancer Settings:**

* Regional, SFO3 (same as your servers)
* Default VPC (same as your servers)
* External (public)
* Use the "web" tag to load balance all servers with a web tag in the SF03 region

![Image of creation for load balancer](/Load_Balancer.png)

A load balancer is a tool to help to distribute incoming traffic among multiple servers to improve a service or application's performance and reliability. Our cloud-based load balancer helps distribute internet traffic evenly between our two servers. The load balancer is using a dynamic algorithm so it's looking at things like server health, their current status, how well they are performing, etc when dsitributing traffic.

![Explanation image for load balancer](/Load_Balancer_Explanation.png)

## Creating the System User and Directory Structure on Both Droplets

1. Create the system user with home directory at /var/lib/webgen and a non-login shell:

`sudo useradd -r -m -d /var/lib/webgen -s /usr/sbin/nologin webgen` 

<br>

**What does this do?**

`-r`: Creates a system user.

`-m`: Creates the home directory if it doesn't already exist.

`-d /var/lib/webgen`: Specifies the home directory for the user.

`-s /usr/sbin/nologin`: Sets a non-login shell to prevent direct login.

<br>

The benefit of creating webgen as a system user because it has limited privileges and is used for running system services or background tasks. Now our user is set up without requiring direct login or unnecessary privileges.

<br>

2. Create the Home Directory Structure

`sudo mkdir /var/lib/webgen/bin`
`sudo mkdir /var/lib/webgen/HTML` 

<br>

3. Move the `generate_index` script into the bin folder

`sudo mv generate_index /var/lib/webgen/bin`

<br>

4. Give Ownership of the Home Directory to Webgen and Make `generate_index` Executable 

`sudo chown -R webgen:webgen /var/lib/webgen`

`sudo chmod 700 /var/lib/webgen/bin/generate_index` 

<br>

## Setting up the `.timer` and `.service` Files for generate-index

1. Move Both the Provided `generate-index.service` and `generate-index.timer` Files to `/etc/systemd/system`

`sudo mv ~/generate-index.service /etc/systemd/system`

`sudo mv ~/generate-index.timer /etc/systemd/system`

<br>

**What does `generate-index.service` do?**

This `.service` file runs the generate_index script located at `/var/lib/webgen/bin/` by the webgen user to genrate an index.html file. We use the index.html file to display content for our web server. 

```
[Unit]
Description=Run generate_index script to create the index.html file
Wants=multi-user.target

[Service]
User=webgen
Group=webgen
ExecStart=/var/lib/webgen/bin/generate_index
Restart=on-failure
Type=oneshot

[Install]
WantedBy=multi-user.target
```

* `Wants=multi-user.target` specifies a soft dependency wants the multi-user.target to be active. `multi-user.target` is the state where multiple users can log in.
* `ExecStart=` to specify the location of the script we're running
* `Type=oneshot` specifies that the service does it's task once and doesn't remain active in the background

<br>

**What does `generate-index.timer` do?**

The `.timer` file makes a systemd timer unit that schedules `generate-index.service` to run every day at 5:00 AM. 

```
[Unit]
Description=Runs the generate-index.service every day at 5:00

[Timer]
OnCalendar=*-*-* 05:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

* `OnCalendar=` sets a reoccuring time for the timer to run

<br>

**Why `/etc/systemd/service`?**

We move the `.service` and `.timer` file to `/etc/systemd/service` because it's the common directory for user-defined or custom service and timer files. It also ensures that these files will persist across updates.

<br>

2. Reload Systemd After the Changes

`sudo systemctl daemon-reload` 

<br>

3. Enable and Start the Timer and Service 

`sudo systemctl enable --now generate-index.timer`

`sudo systemctl enable --now generate-index.service`

<br>

4. Verify the Timer and Service Run Successfully

- To check when the timer last ran and when it will next run: `systemctl list-timers --all | grep generate-index.timer`

- To check if the service ran successfully: `sudo systemctl status generate-index.service`

- View logs for generate-index.service: `sudo journalctl -u generate-index.service`

- View logs for the generate-index.timer: `sudo journalctl -u generate-index.timer`

<br>

## Setting Up `nginx`

Nginx is a webserver. Web servers are software that we can run on our hardware server. The purpose of which is serve documents over the internet. Generally using HTTP(s). Web servers help to provide secure access to documents. Many provide additional features like load balancing and proxy forwarding.

1. Download nginx

`sudo pacman -S nginx`

<br>

1. Move the `nginx.conf` file 

`sudo mv ~/nginx.conf /etc/nginx/`

```
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
include sites-enabled/*;
```
These are to load additional configuration files to extend or modify nginx's behavior.

* `include /etc/nginx/sites-enabled/*;` and `include sites-enabled/*;` These include all files in the sites-enabled/ directory and is commonly used to enable virtual hosts. 

<br>

3. Make a documents directory and create the text files

`sudo mkdir /var/lib/webgen/documents`

`sudo nvim /var/lib/webgen/documents/file-one`

`sudo nvim /var/lib/webgen/documents/file-two`

<br>

3. Make folders for the server block 

`sudo mkdir /etc/nginx/sites-available`

`sudo mkdir /etc/nginx/sites-enabled`

<br>

We created these two folders based on the template for setting up nginx given in the ArchiLinux wiki. They are a common convention in the configuration of nginx. This structure is used to manage our server block configuration.



<br>

4. Add the provided server block file `sites.conf` to `/etc/nginx/sites-available`

`sudo mv ~/sites.conf /etc/nginx/sites-available`

<br>

We created a seperate server block file instead of putting it in nginx.conf because it's easier to maintain. We can add sites by creating new server block files into the `/etc/nginx/sites-available` folder and creating symbolic links for them. 

<br>

**Inside our `sites.conf` server block** 

```
server {
	listen 80;
	listen [::]:80;

	server_name _;

	root /var/lib/webgen/HTML;
	index index.html;

	location /documents {
  		alias /var/lib/webgen/documents/;
		autoindex on;
	} 

	location / {
		try_files $uri $uri/ =404;
	}
}
```

* `listen 80` tells nginx to listen for incoming HTTP requests on port 80
* `root /var/lib/webgen/HTML` Sets the main folder for where our website's files are stored. When someone visits the site, nginx will look in this folder to find and serve the requested files.
* `alias /var/lib/webgen/documents/;` incoming requests to /documents in your website's URL will be associate to the folder /var/lib/webgen/documents/ on the server. It’s used when the files you want to serve for a specific URL path aren’t in the main website folder.
* Root uses the full location path as part of the file lookup.It appends the URL path to the root value to find the file.
* Alias maps the URL path to the specified directory but it doesn't append the location path.

5. Create a symbolic link 

`sudo ln -s /etc/nginx/sites-available/sites.conf /etc/nginx/sites-enabled/sites.conf`

<br>

6. Reload Systemd After the Changes

`sudo systemctl daemon-reload` 

<br>

7. Use `sudo nginx -t` to check for errors in the nginx configuration. 

<br>

8. Start and enable nginx

`sudo systemctl start nginx`

`sudo systemctl enable nginx`

<br>

- Use `sudo systemctl status nginx` to check the nginx status and verify it's running. 
- Use `sudo systemctl reload nginx` to reload nginx to apply any changes you've made to the configuration.
- Use `sudo nginx -t` to check for errors in the nginx configuration. 

## Setting Up `ufw`

1. Download ufw

`sudo pacman -S ufw`

<br>

1. Allow SSH connection in our firewall

`sudo ufw allow ssh`

* allows incoming SSH connections by enabling the default SSH port 22
<br>

2. Limit the rate for ssh connections

`sudo ufw limit ssh`

* Limits the rate of SSH connections to prevent brute-force attacks

<br>

3.  Allow http connections

`sudo ufw allow http`

* Allows incoming HTTP connections by opening the default HTTP port 80

<br>

4. Reload after changes

`sudo systemctl daemon-reload`

<br>

5. Enable the firewall

`sudo ufw enable`

`sudo systemctl enable --now ufw.service`

<br>

6. Check the status of the firewall to confirm everything is working correctly

`sudo ufw status verbose`

* Displays the current status of the UFW firewall along with detailed information about active rules and configurations

<br>

## Your Web Server is Now Set Up!

With everything complete your final page should look like this: 

![Image of system information on webpage](/Success_Screenshot.png)









