---
id: tutorial-email-server
summary: Learn how to set up postfix and dovecot in order to host an imap or pop3 email server.
categories: server
tags: tutorial, email, dovecot, postfix, server, guide, imap, pop3
difficulty: 4
status: published
feedback_url: https://github.com/canonical-websites/tutorials.ubuntu.com/issues
published: 2017-12-17
author: Varun Patel <varun-patel@live.com>

---

# Install A VPN Server Using Postfix and Dovecot

## Overview
Duration: 3:00

In this tutorial, we will go over the installation and configuration of Postfix and Dovecot to create either an imap or pop3 email server.
We will begin by installing all the programs necessary: MySQL, Postfix and Dovecot. We will then configure each of these.

### What You'll Learn:

* How to install `mail-stack-delivery`
* How to install and configure MySQL
* How to install and configure Postfix
* How to install and configure Dovecot

### What You'll Need

* A computer running Ubuntu 16.04 Xenial Xerus or above
* Updated apt package lists (you can do this in terminal using `sudo apt-get update`)
* An intermediate understanding of how email and email servers work
* To be comfortable using terminal
* A stable internet connection.

Survey
: How will you use this tutorial?
 - Only read through it
 - Read it and complete the exercises
: What is your current level of experience?
 - Novice
 - Intermediate
 - Proficient

## Installing the Required programs
Duration: 5:00

Lets start with MySQL:
```bash
sudo apt-get install mysql-server
```
When propmted to enter a password, ensure that you do so.
Now we need to set up the FQDN we will be using.
```bash
sudo nano /etc/hosts
```
And edit this file with the following
```bash
127.0.0.1	localhost.localdomain	  localhost
127.0.1.1       hostname.yourdomain.tld	  hostname
yourip  	hostname.yourdomain.tld	  hostname
```
Replace the following:
* *HOSTNAME* --> your hostname
* *YOURDOMAIN* --> your domain
* *TLD* --> your top level domain
* *YOURIP* --> your ip address

There are six packages we need to install, they can be installed by:
```bash
sudo apt-get install mail-stack-delivery
```

We can now configure MySQL click next to continue.

## Configuring MySQL
Duration: 7:00

We need to create three MySQL Tables:
* Domains
* Users
* Aliases

First create the mailserver database by entering root and creating a database:
```bash
sudo -i
mysqladmin -p create mailserver
sudo -i -u $USER
```
Replace $USER with your username.

Now we can enter the MySQL root user
```bash
mysql -u root -p
```
If this is successful your prompt should look like `mysql >`

We need to create a mailserver user and grant permissions to it:
```bash
GRANT SELECT ON mailserver.* TO 'usermail'@'127.0.0.1' IDENTIFIED BY 'mailpassword';
FLUSH PRIVILEGES;
```

### Creating the Database

Now we must tell MySQL to use the database `mailserver`
```bash
USE mailserver;
```

Now we can create a domains table:
```bash
CREATE TABLE `virtual_domains` (
`id`  INT NOT NULL AUTO_INCREMENT,
`name` VARCHAR(50) NOT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

Another one for users:
```bash
CREATE TABLE `virtual_users` (
`id` INT NOT NULL AUTO_INCREMENT,
`domain_id` INT NOT NULL,
`password` VARCHAR(106) NOT NULL,
`email` VARCHAR(120) NOT NULL,
PRIMARY KEY (`id`),
UNIQUE KEY `email` (`email`),
FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

And one last one for the aliases:
```bash
CREATE TABLE `virtual_aliases` (
`id` INT NOT NULL AUTO_INCREMENT,
`domain_id` INT NOT NULL,
`source` varchar(100) NOT NULL,
`destination` varchar(100) NOT NULL,
PRIMARY KEY (`id`),
FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### Populating the Database

First we can populate the domains table:
```bash
INSERT INTO `mailserver`.`virtual_domains`
(`id` ,`name`)
VALUES
('1', 'yourdomain.tld'),
('2', 'hostname.yourdomain.tld');
```
Be sure to replace example.tld and hostname.example.tld with your actual FQDN.

Now for the users table:
```bash
INSERT INTO `mailserver`.`virtual_users`
(`id`, `domain_id`, `password` , `email`)
VALUES
('1', '1', ENCRYPT('secure_password_1', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'email1@yourdomain.tld'),
('2', '1', ENCRYPT('secure_password_2', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'email2@yourdomain.tld');
```

Ensure you replace the following with your own values:
* *secure_password_1* & *secure_password_2*
* *email1@yourdomain.tld* & *email2@yourdomain.tld*

Finally we can populate the aliases table:
```bash
INSERT INTO `mailserver`.`virtual_aliases`
(`id`, `domain_id`, `source`, `destination`)
VALUES
('1', '1', 'alias@example.com', 'email1@yourdomain.tld');
```

Make sure to change the values to fit your server.

Now type exit to leave the MySQL Root prompt

## Configuring Postfix
Duration: 15:00

We are now able to configure postfix to handle smtp connections.

First, we must edit the main configuation file:
```bash
sudo nano /etc/postfix/main.cf
```
