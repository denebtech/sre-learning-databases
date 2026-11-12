# MySQL server

## Lab 1: Creating MySQL server

This lab creates a Debian-based VM with MariaDB, configures the root password, and creates a sample database and user for application access.

### 1. Start the VM

From this directory, bring up the virtual machine:

```bash
vagrant up
```

This uses the `generic/debian12` box and creates a VM with:

- hostname: `mysql-server`
- private IP: `192.168.56.11`
- MySQL port mapped from guest `3306` to host `3306`

### 2. Provision the database with Ansible

Run the playbook to install MariaDB and configure the database:

```bash
ansible-playbook -i inventory.ini playbook.yaml
```

### 3. Verify the service

Check that MariaDB is running:

```bash
vagrant ssh -c "sudo systemctl status mariadb"
```

You can also test connectivity from the host:

```bash
mysql -u root -p
```

### 4. Validate the database objects

Once connected to MariaDB, verify that the database and user were created:

```sql
SHOW DATABASES;
USE lab_db;
SHOW TABLES;
SELECT user, host FROM mysql.user;
```

### Summary

This lab demonstrates the basic steps for provisioning a MySQL-compatible database service with Vagrant and Ansible:

1. create a VM
2. install MariaDB
3. secure the root account
4. create a database and application user
5. grant permissions for application access

---

You can stop the VM after the lab with:

```bash
vagrant halt
```
