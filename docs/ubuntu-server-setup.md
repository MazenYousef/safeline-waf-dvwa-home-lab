# Ubuntu Web Server Setup: LAMP, DVWA and SafeLine WAF

Configuration of the Ubuntu 24.04 LTS VM (`192.168.56.104`), which hosts the vulnerable application and the WAF in front of it.

[← Back to the main README](../README.md)

> ⚠️ The credentials below are lab-only values for an isolated, intentionally vulnerable setup. Never reuse them anywhere else.

## 1. Update the system and install the LAMP stack

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y net-tools openssl
sudo apt-get install -y apache2 php php-mysql mysql-server
```

Secure the MySQL installation by setting the root password policy, removing anonymous users, and removing the test database:

```bash
sudo mysql_secure_installation
```

## 2. Install DVWA

```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
sudo chown -R www-data:www-data DVWA
sudo chmod -R 755 DVWA
```

## 3. Configure the DVWA database

### 3.1 Create the database and a dedicated user

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE dvwa;
CREATE USER 'dvwa_user'@'localhost' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL ON dvwa.* TO 'dvwa_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 3.2 Point DVWA to the database

DVWA ships a template config. Copy it, then edit it:

```bash
cd /var/www/html/DVWA/config
sudo cp config.inc.php.dist config.inc.php
sudo nano config.inc.php
```

```php
$_DVWA[ 'db_server' ]   = 'localhost';
$_DVWA[ 'db_database' ] = 'dvwa';
$_DVWA[ 'db_user' ]     = 'dvwa_user';
$_DVWA[ 'db_password' ] = 'p@ssw0rd';
$_DVWA[ 'db_port' ]     = '3306';
// DBMS: MySQL
```

The database user only has rights on the `dvwa` database. That limits the damage if the application is compromised, which is the principle of least privilege.

### 3.3 Initialise DVWA

Open `http://192.168.56.104/DVWA/setup.php` and click **Create / Reset Database**. Apache is still on port 80 at this point.

## 4. Add custom test data

These sample accounts give the lab realistic data to target with SQL injection:

```bash
sudo mysql -u root -p
```

```sql
USE dvwa;

CREATE TABLE test_users (
  id INT NOT NULL AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  password VARCHAR(50) NOT NULL,
  PRIMARY KEY (id)
);

INSERT INTO test_users (username, password) VALUES
  ('alice', 'alice123'),
  ('bob',   'bob123'),
  ('admin', 'admin123');

EXIT;
```

The passwords are deliberately stored in plain text to show what an attacker could extract. A real application must store only salted password hashes.

## 5. Move Apache to port 8080

SafeLine runs on the same VM and needs ports 80 and 443, so Apache becomes the backend on **8080**.

`/etc/apache2/ports.conf`:

```apache
Listen 8080
```

`/etc/apache2/sites-available/000-default.conf`:

```apache
<VirtualHost *:8080>
    ...
</VirtualHost>
```

Restart Apache and confirm the port:

```bash
sudo systemctl restart apache2
sudo ss -tlnp | grep apache2     # should show :8080
```

> **Hardening tip:** `Listen 8080` accepts connections from the whole Host-Only network, so DVWA can still be reached directly on `http://192.168.56.104:8080` and bypass the WAF. In production, allow the backend port only from the WAF, for example with `Listen 127.0.0.1:8080` or a firewall rule.

## 6. Install SafeLine WAF

Run the official installer, which also installs Docker if needed. The `--en` flag selects English:

```bash
sudo bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
```

Project page: <https://cyberserval.tech/landing/safeline>

When it finishes, the installer prints the dashboard address (port **9443** on the VM) and the admin credentials. Store them safely, and never commit them.

Continue with the certificate and SafeLine configuration in the [main README](../README.md#4-self-signed-tls-certificate).
