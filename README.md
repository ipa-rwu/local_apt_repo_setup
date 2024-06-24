## Set up local apt repository using Apache HTTP Server

### Step 1: Install Apache HTTP Server
If Apache is not already installed, you can install it using the following command:
```bash
sudo apt update
sudo apt install apache2
```
### Step 2: Create your Repository
* Create a folder to host server, such as ` /var/www/html/326_apt_repo`
```
mkdir -p /var/www/html/326_apt_repo/pool/main/
```
we will create a different directory to contain a list of all available packages and corresponding metadata:
```
mkdir -p /var/www/html/326_apt_repo/dists/stable/main/binary-amd64
```
### Step 2: Configure Apache to Serve Your Repository
* Create a folder to host server, such as ` /var/www/html/326_apt_repo`
  You'll need to configure Apache to serve files from ` /var/www/html/326_apt_repo`. We’ll do this by setting up a new virtual host.
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
    DocumentRoot /var/www/html/326_apt_repo
    ServerName 192.168.0.214

    <Directory /var/www/html/326_apt_repo>
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
Ensure that the Apache user (usually www-data) has read access to  /var/www/html/326_apt_repo. If your repository directory is under a home directory, it’s also important to make sure that the entire path is accessible to the Apache user.
```
sudo chmod -R 755 /var/www/html/326_apt_repo
sudo chown -R www-data:www-data /var/www/html/326_apt_repo
```

### Step 4: Add Packages and Generate Repository Metadata
Now, populate your repository with Debian packages and generate the metadata as described in previous steps.

Populate the Repository
Copy your .deb files into  /var/www/html/326_apt_repo.

* Generate Metadata
  You can use the dpkg-scanpackages tool to generate the necessary metadata. First, install the required package if not already installed:
  ```
  sudo apt install dpkg-dev
  ```
Next, we will generate a Packages file, which will contain a list of all available packages in this repository. We will use the dpkg-scanpackages program to generate it, by running:
```
 cd  /var/www/html/326_apt_repo
dpkg-scanpackages --arch amd64 pool/ > dists/stable/main/binary-amd64/Packages
```
It’s also good practice to compress the packages file, as apt will favour downloading compressed data whenever available.
  ```
  cd  /var/www/html/326_apt_repo
  cat dists/stable/main/binary-amd64/Packages | gzip -9 > dists/stable/main/binary-amd64/Packages.gz
  ```

### Step 5: Secure Your Repository
For additional security, especially if your repository is public, you can sign your repository using GPG:

**Create a new Public/Private PGP Keypair**
```
echo "%echo Generating an example PGP key
Key-Type: RSA
Key-Length: 2048
Name-Real: 326_apt_repo
Name-Email: example@example.com
Expire-Date: 0
%no-ask-passphrase
%no-protection
%commit" > /tmp/326_apt_repo-pgp-key.batch
```
then we will generate it under a new temporary gpg keyring:
```
export GNUPGHOME="$(mktemp -d /var/www/html/326_apt_repo/pgpkeys-XXXXXX)"
gpg --no-tty --batch --gen-key /tmp/326_apt_repo-pgp-key.batch
```
Since we overrode the GNUPGHOME to a temporary directory, we can keep this key separate from our other keys. Let’s take a quick look at the contents of the directory:

```
ls "$GNUPGHOME/private-keys-v1.d"
```
We can also view all of our loaded keys with:
```
gpg --list-keys
```
This PGP key is comprised of both a public key, and a private key. Let’s start with exporting the public key:
```
gpg --armor --export 326_apt_repo > /var/www/html/326_apt_repo/pgp-key.public
```
You can verify only a single key was exported by running:
```
cat /var/www/html/326_apt_repo/pgp-key.public | gpg --list-packets
```
Next let’s export the private key so we can back it up somewhere safe.
```
gpg --armor --export-secret-keys 326_apt_repo > /var/www/html/326_apt_repo/pgp-key.private
```

**Sign Your Repository**
Generate a Release file which contains an index of your repository metadata:
```
apt-ftparchive release . > Release
```
Before we start signing with out keys, let’s make sure that we can import the backup we made. To do that, we will create a new GPG keyring location:
```
export GNUPGHOME="$(mktemp -d /var/www/html/326_apt_repo/pgpkeys-XXXXXX)"
```
and then verify that it is empty:
```
gpg --list-keys
```
Next we will import our backed up private key:
```
cat /var/www/html/326_apt_repo/pgp-key.private | gpg --import
```
let’s get around to signing the Release file now.
```
cat /var/www/html/326_apt_repo/dists/stable/Release | gpg --default-key 326_apt_repo -abs > ~/326_apt_repo/dists/stable/Release.gpg
```
we will create a third file InRelease which will combine both the contents of Release and the PGP signature:
```
cat /var/www/html/326_apt_repo/dists/stable/Release | gpg --default-key 326_apt_repo -abs --clearsign > ~/326_apt_repo/dists/stable/InRelease
```

### Step 6: Use the Repository on Client Machines
Clients need to add your repository to their list of sources:

```
echo "deb [arch=amd64 signed-by=/var/www/html/326_apt_repo/pgp-key.public] http://192.168.0.214/326_apt_repo stable main" | sudo tee /etc/apt/sources.list.d/326_apt_repo.list
sudo apt update
```
Now, they can install packages from your repository using apt install.

