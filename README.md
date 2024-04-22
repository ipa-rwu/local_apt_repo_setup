## Set up local apt repository using Apache HTTP Server

### Step 1: Install Apache HTTP Server
If Apache is not already installed, you can install it using the following command:
```bash
sudo apt update
sudo apt install apache2
```

### Step 2: Configure Apache to Serve Your Repository
* Create a folder to host server, such as `/home/"user name"/326_apt_repo`
  You'll need to configure Apache to serve files from `/home/"user name"/326_apt_repo`. We’ll do this by setting up a new virtual host.
* Create a Virtual Host Configuration
  Create a new configuration file in the Apache sites-available directory:
  ```
  sudo nano /etc/apache2/sites-available/326_apt_repo.conf
  ```
  
* Add the following configuration to the file. This setup tells Apache to serve files from your specified directory and allows HTTP traffic to it:

  You need to change servername to current server IP
  ```
  <VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /home/"user name"/326_apt_repo
    ServerName 192.168.0.214

    <Directory /home/"user name"/326_apt_repo>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/326_apt_repo_error.log
    CustomLog ${APACHE_LOG_DIR}/326_apt_repo_access.log combined
   </VirtualHost>
  ```

* Enable the New Site
  ```
  sudo a2ensite 326_apt_repo.conf
  sudo systemctl reload apache2
  ```

### Step 3: Adjust Permissions
Ensure that the Apache user (usually www-data) has read access to /home/"user name"/326_apt_repo. If your repository directory is under a home directory, it’s also important to make sure that the entire path is accessible to the Apache user.
```
sudo chmod -R 755 /home/"user name"/326_apt_repo
sudo chown -R www-data:www-data /home/nuc/326_apt_repo
```

### Step 4: Add Packages and Generate Repository Metadata
Now, populate your repository with Debian packages and generate the metadata as described in previous steps.

Populate the Repository
Copy your .deb files into /home/"user name"/326_apt_repo.

* Generate Metadata
  You can use the dpkg-scanpackages tool to generate the necessary metadata. First, install the required package if not already installed:
  ```
  sudo apt install dpkg-dev
  ```

  This command creates a Packages.gz file that APT uses to get information about available packages.
  ```
  cd /home/nuc/326_apt_repo
  dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz
  ```

### Step 5: Secure Your Repository
For additional security, especially if your repository is public, you can sign your repository using GPG:

**Create a GPG Key**
```
gpg --full-gen-key
```
Follow the prompts to generate a key. Remember the key ID, you will need it to sign your repository.

**Sign Your Repository**
Generate a Release file which contains an index of your repository metadata:
```
apt-ftparchive release . > Release
```

Now sign the Release file using your GPG key:
```
gpg --default-key "your-key-id" -abs -o Release.gpg Release
```

**Distribute Your Public Key**
Distribute your GPG public key so that users can verify the signatures:

```
gpg --export -a "your-key-id" > my-repo-key.asc
```

Users will need to add this key to their system to use your repository:

```
Copy code
sudo apt-key add my-repo-key.asc
```

### Step 6: Use the Repository on Client Machines
Clients need to add your repository to their list of sources:

```
echo 'deb http://"server IP"/ /' | sudo tee /etc/apt/sources.list.d/326-repo.list
sudo apt update
```
Now, they can install packages from your repository using apt install.
