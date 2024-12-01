## Prerequisites 
- A clone of the provided Git repository 
- nginx and ufw packages installed

<br>

## Create Your Droplets and Load Balancer

1. Create your 2 droplets on digital ocean with the tag "web"

* Refer to digitalOcean documentation if you need help

<br>

2. Create a load balancer for the droplets

* Refer to digitalOcean documentation if you need help

**Load Balancer Settings:**

* Regional, SFO3 (same as your servers)
* Default VPC (same as your servers)
* External (public)
* Use the "web" tag to load balance all servers with a web tag in the SF03 region

![Image of creation for load balancer](/Load_Balancer.png)

A load balancer is a tool to help to distribute traffic among multiple servers to improve a service or application's performance and reliability. 

## Creating the System User and Directory Structure on Both Droplets

1. Create the system user with home directory at /var/lib/webgen and a non-login shell:

`sudo useradd -r -m -d /var/lib/webgen -s /usr/sbin/nologin webgen` 

<br>

**What does this do?**

`-r`: Creates a system user.

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

1. Download nginx

<br>

1. Move the `nginx.conf` file 

`sudo mv ~/nginx.conf /etc/nginx/`

<br>

3. Make a documents folder and create the text files

`sudo mkdir /var/lib/webgen/documents`

`sudo nvim /var/lib/webgen/documents/file-one`

`sudo nvim /var/lib/webgen/documents/file-two`

3. Make folders for the server block 

`sudo mkdir /etc/nginx/sites-available`

`sudo mkdir /etc/nginx/sites-enabled`

<br>

We created these two folders based on the template for setting up nginx given in the ArchiLinux wiki. They are a common convention in the configuration of nginx. This structure is used to manage our server block configuration.

<br>

4. Add the provided server block file `sites.conf` to `/etc/nginx/sites-available`

`sudo mv ~/sites.conf /etc/nginx/sites-available`

<br>

**Inside our `sites.conf`**

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
* `root /var/lib/webgen/HTML` specifies where to find the document's root (the directory that contains the website’s files). When a request is made to the server, nginx will serve files from this directory
* `alias /var/lib/webgen/documents/;` Requests to `/documents` are mapped to `/var/lib/webgen/documents/`. The alias tells nginx to look at the file path `/var/lib/webgen/documents/` when someone goes to the url `/documents`. Alias allows you to serve files from a directory that is different from the root of your website.

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

We created a seperate server block file instead of putting it in nginx.conf because it's easier to maintain. We can add sites by creating new server block files into the `/etc/nginx/sites-available` folder and creating symbolic links. 

- Use `sudo systemctl status nginx` to check the nginx status and verify it's running. 
- Use `sudo systemctl reload nginx` to reload nginx to apply any changes you've made to the configuration.
- Use `sudo nginx -t` to check for errors in the nginx configuration. 

## Setting Up `ufw`

1. Download ufw

`sudo pacman -S ufw`

<br>

1. Allow SSH connection in our firewall

`sudo ufw allow ssh`

<br>

2. Limit the rate for ssh connections

`sudo ufw limit ssh`

<br>

3.  Allow http connections

`sudo ufw allow http`

<br>

4. Enable the firewall

`sudo systemctl enable --now ufw.service`

<br>

5. Check the status of the firewall to confirm everything is working correctly

`sudo ufw status verbose`

<br>

## Your Web Server is Now Set Up!

With everything complete your final page should look like this: 

![Image of system information on webpage](/Success_Screenshot.png)









