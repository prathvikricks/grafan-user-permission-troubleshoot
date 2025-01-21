# Documentation: Adding an Admin User to Grafana

This guide explains how to add an administrator user to a Grafana instance running inside a Docker container by interacting directly with the `grafana.db` SQLite database. This method is helpful when you cannot access the Grafana UI or API.

---

## **Step 1: Access the Grafana Docker Container**
1. Identify the Grafana container:
   ```bash
   docker ps
   ```
   Locate the container running Grafana in the list.

2. Access the container:
   ```bash
   docker exec -it <container_name> /bin/bash
   ```
   Replace `<container_name>` with the name of the Grafana container.

---

## **Step 2: Install SQLite Inside the Container**
If the SQLite3 CLI tool is not already installed in the container, install it:

- For **Debian-based images**:
  ```bash
  apt update && apt install sqlite3 -y
  ```
- For **Alpine-based images**:
  ```bash
  apk add sqlite
  ```

---

## **Step 3: Connect to the Grafana Database**
1. Navigate to the directory where `grafana.db` is stored:
   ```bash
   cd /var/lib/grafana
   ```

2. Open the SQLite database:
   ```bash
   sqlite3 grafana.db
   ```

---

## **Step 4: Inspect the `user` Table Schema**
To understand the table structure, run:
```sql
PRAGMA table_info(user);
```
This will display the columns and constraints for the `user` table. Example output:
```
0|id|INTEGER|1||1
1|version|INTEGER|1||0
2|login|TEXT|1||0
3|email|TEXT|1||0
4|name|TEXT|0||0
5|password|TEXT|0||0
6|salt|TEXT|0||0
7|rands|TEXT|0||0
8|company|TEXT|0||0
9|org_id|INTEGER|1||0
10|is_admin|INTEGER|1||0
11|email_verified|INTEGER|0||0
12|theme|TEXT|0||0
13|created|DATETIME|1||0
14|updated|DATETIME|1||0
15|help_flags1|INTEGER|1|0|0
16|last_seen_at|DATETIME|0||0
17|is_disabled|INTEGER|1|0|0
18|is_service_account|BOOLEAN|0|0|0
19|uid|TEXT|0||0
```

---

## **Step 5: Add a New Admin User**
Use the following SQL command to add a new admin user:
```sql
INSERT INTO user (
    version, login, email, name, password, salt, rands, company,
    org_id, is_admin, email_verified, theme, created, updated,
    help_flags1, last_seen_at, is_disabled, is_service_account, uid
)
VALUES (
    0, 'admin', 'admin@example.com', 'Admin User',
    '8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918',
    '', '', '', 1, 1, 0, '',
    CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 0, CURRENT_TIMESTAMP,
    0, 0, 'admin-uid'
);
```
### **Explanation of Values**:
- `login`: Username of the admin (e.g., `admin`).
- `email`: Email address for the user (e.g., `admin@example.com`).
- `password`: SHA-256 hash of the password (in this case, the hash corresponds to "admin").
- `is_admin`: Set to `1` to grant admin privileges.
- `org_id`: Set to `1` for the default organization.

---

## **Step 6: Verify the New User**
To confirm that the user was added successfully, run:
```sql
SELECT * FROM user WHERE login = 'admin';
```

Example output:
```
3|0|admin|admin@example.com|Admin User|8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918||||1|1|0||2025-01-21 14:43:35|2025-01-21 14:43:35|0|2025-01-21 14:43:35|0|0|admin-uid
```

---

## **Step 7: Restart the Grafana Container**
After making changes to the database, restart the container to apply the updates:
```bash
docker restart <container_name>
```

---

## **Step 8: Log in to Grafana**
Use the credentials you created to log in:
- **Username**: `admin`
- **Password**: `admin`

If the login fails, verify the password hash and username in the database.

---

## **Step 9: Update Password (Optional)**
Once logged in, immediately update the password through the Grafana UI:
1. Go to **Profile** > **Change Password**.
2. Set a strong, secure password.

---

## **Additional Notes**
- Direct database manipulation is risky. Prefer using Grafana CLI or API whenever possible.
- To reset the admin password via CLI:
  ```bash
  grafana-cli admin reset-admin-password <newpassword>
  ```

---

## Grafana Troubleshooting and Resolution Documentation

### Problem Overview
While managing a Grafana instance using Docker, the following issues were encountered:

1. **Admin User Reset Error:**
   - Attempting to reset the admin password resulted in an error:
     ```
     Admin user cannot be found.

     Please try to run the command again specifying a user-id (--user-id) from the list below:
     Username: 1717 ID: 2
     Username: admin ID: 3
     ```

2. **Dashboard Invisibility:**
   - Existing dashboards could not be seen in the UI despite being present in the database.

3. **User Permissions Issue:**
   - A user with Viewer permissions could not access all dashboards or administrative functionalities.

### Troubleshooting Steps

#### Step 1: Identify Available Users
To address the admin reset issue, we first listed all users in the Grafana instance:

```bash
docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "SELECT id, login, email FROM user;"
```
This confirmed the presence of multiple users:
- ID: 2, Login: 1717
- ID: 3, Login: admin

#### Step 2: Reset Admin Password
Using the user ID for the admin account, the password was reset with the following command:

```bash
docker exec -it grafana grafana-cli admin reset-admin-password --user-id 3 admin
```
This successfully reset the password to `admin` for the admin user.

#### Step 3: Verify Dashboards in the Database
To ensure dashboards were still present, we queried the database:

```bash
docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "SELECT id, uid, title FROM dashboard;"
```
Output:
- Production Metrics
- Development Metrics
- Production logs
- Development logs
- Production ASG logs
- Prod ASG metrics

This confirmed that the dashboards were intact in the database.

#### Step 4: Check Dashboard Permissions
To identify access issues, we checked the dashboard permissions:

```bash
docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "SELECT dashboard_id, user_id, permission FROM dashboard_acl;"
```
Output:
- `dashboard_id = -1` with default permissions:
  - `permission = 1` (View)
  - `permission = 2` (Edit)

This indicated global permissions were set, but user-specific roles were required for visibility.

#### Step 5: Verify User Roles
We checked the roles assigned to users in the organization:

```bash
docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "SELECT org_user.user_id, user.login, org_user.role FROM org_user JOIN user ON org_user.user_id = user.id;"
```
Output:
- User ID: 2, Login: 1717, Role: Viewer

The Viewer role restricted access to dashboards and administrative features.

#### Step 6: Update User Role
To provide full administrative access, we updated the role for user ID 2:

```bash
docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "UPDATE org_user SET role = 'Admin' WHERE user_id = 2;"
```
After logging out and back in, the user now had full access.

#### Step 7: Locate and Update Custom User
To address concerns about a specific custom user, we listed all users and their details:

```bash
docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "SELECT id, login, email FROM user;"
```
The custom user was identified, and their role was updated similarly:

```bash
docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "UPDATE org_user SET role = 'Admin' WHERE user_id = <custom_user_id>;"
```

### Root Cause Analysis
The problems stemmed from:
1. Multiple admin-level users in the system requiring explicit identification for password resets.
2. Role misconfiguration causing restricted access to dashboards and administrative features.
3. Viewer role limitation for user ID 2 (1717).

### Solutions Implemented
1. Reset the password for the primary admin user using their specific user ID.
2. Updated roles to provide administrative access for required users.
3. Verified dashboard integrity and permissions in the database.

### Results
- Admin user password was successfully reset.
- Dashboards became visible and accessible after correcting user roles.
- Full administrative functionality was restored to relevant users.

### Recommendations
1. Maintain clear documentation of user roles and permissions.
2. Regularly back up the Grafana database to avoid loss of configurations.
3. Use persistent volume mounts to retain data during container recreation.

### Additional Commands for Future Reference
- **List all users:**
  ```bash
  docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "SELECT id, login, email FROM user;"
  ```
- **View user roles:**
  ```bash
  docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "SELECT org_user.user_id, user.login, org_user.role FROM org_user JOIN user ON org_user.user_id = user.id;"
  ```
- **Update user role:**
  ```bash
  docker exec -it grafana sqlite3 /var/lib/grafana/grafana.db "UPDATE org_user SET role = 'Admin' WHERE user_id = <user_id>;"
  ```

This documentation ensures effective troubleshooting and recovery of Grafana configurations in the future.

