# Practical 6: Securing Redis and MongoDB
## Authentication, Encryption, RBAC, and Security Audit

**Module:** DBS302 - NoSQL Database Management  
**Student:** Pema Tshering Yangchen  
**Student ID:** 2230295  
**Date:** 4 May 2026  

---

## 1. Aim

Set up and confirm Auth., Encyption, and Role Based Access to Redis and MongoDB, as well as run a generic security audit on configured DBs.

---

## 2. Objectives

Upon completion of this practical, the student will be able to:

- Configure password-based authentication; create access control lists (ACLs) on Redis.
- Configure Redis for TLS Encryption.
- Configure user authentication on MongoDB; enable role-based access control (RBAC) in MongoDB using both built-in and custom-defined user roles.
- Enable TLS on MongoDB to encrypt network traffic between all clients and MongoDB in-transit.
- Complete a security audit checklist for both DBs; document audit results roles.

---

## 3. Theory

### 3.1 Authentication

Authentication is the process of verifying the identity of a client before granting access to the database. In Redis, this is achieved through ACL users with usernames and passwords. In MongoDB, authentication is enabled by setting `authorization: enabled` in `mongod.conf`, after which all clients must provide valid credentials.

### 3.2 Encryption (TLS)

Transport Layer Security (TLS) encrypts data as it travels between the client and the server, preventing attackers from intercepting or reading the data in transit. Both Redis and MongoDB support TLS configuration using certificate files. In this practical, self-signed certificates were generated using OpenSSL and used to secure connections on both databases.

### 3.3 Role-Based Access Control (RBAC)

RBAC is a security principle where users are granted only the minimum permissions needed to perform their tasks. In Redis, this is implemented through ACL rules that restrict which commands and key patterns a user can access. In MongoDB, custom roles can be created with specific privileges limited to particular databases and collections, ensuring that application users cannot access administrative or unrelated data.

---

## Table of Contents

### Part A - Securing Redis
- [4.1 Step 1: Verify Redis Installation and Start Server](#41-step-1-verify-redis-installation-and-start-server)
- [4.2 Step 2: Configure ACL Users in redis.conf](#42-step-2-configure-acl-users-in-redisconf)
- [4.3 Step 3: Test ACL Users](#43-step-3-test-acl-users)
- [4.4 Step 4: Generate TLS Certificates for Redis](#44-step-4-generate-tls-certificates-for-redis)
- [4.5 Step 5: Configure Redis for TLS](#45-step-5-configure-redis-for-tls)
- [4.6 Step 6: Test TLS Connection to Redis](#46-step-6-test-tls-connection-to-redis)

### Part B - Securing MongoDB
- [5.1 Step 1: Start MongoDB and Create Root Admin User](#51-step-1-start-mongodb-and-create-root-admin-user)
- [5.2 Step 2: Enable Authentication in mongod.conf](#52-step-2-enable-authentication-in-mongodconf)
- [5.3 Step 3: Verify Authentication Enforcement](#53-step-3-verify-authentication-enforcement)
- [5.4 Step 4: Create Application Role and User (RBAC)](#54-step-4-create-application-role-and-user-rbac)
- [5.5 Step 5: Test RBAC for appUser](#55-step-5-test-rbac-for-appuser)
- [5.6 Step 6: Generate TLS Certificates for MongoDB](#56-step-6-generate-tls-certificates-for-mongodb)
- [5.7 Step 7: Configure MongoDB for TLS](#57-step-7-configure-mongodb-for-tls)
- [5.8 Step 8: Test TLS Connection to MongoDB](#58-step-8-test-tls-connection-to-mongodb)


## 4. Part A - Securing Redis

### 4.1 Step 1: Verify Redis Installation and Start Server

Redis version was confirmed using:

```bash
redis-server --version
```

Redis was started and connectivity was verified:

```bash
redis-cli ping
# Output: PONG
```

### 4.2 Step 2: Configure ACL Users in redis.conf

The following ACL rules were added to `/etc/redis/redis.conf`:

```
user default off
user admin on >adminStrongPwd ~* +@all
user app_user on >appStrongPwd ~session:* +get +set +del +expire +ttl +@connection
user monitoring on >monitorPwd ~* +@read +info +dbsize +lastsave +@connection
```

Redis was restarted to apply the changes:

```bash
sudo systemctl restart redis-server
```

**Explanation of ACL rules:**

- `user default off` - Disables the default unauthenticated user, forcing all clients to authenticate.
- `admin` - Full access to all keys and all commands.
- `app_user` - Restricted to keys matching `session:*` and only basic read/write commands.
- `monitoring` - Read-only access to all keys plus server monitoring commands, but no write access.

### 4.3 Step 3: Test ACL Users

#### Admin User Test

```bash
redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379
```

![alt text](img/1.png)

Commands run:

```
set mykey "hello"   -> OK
get mykey           -> "hello"
```

![alt text](img/2.png)

The admin user has full read/write access to all keys as expected.

#### app_user Test

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379
```

![alt text](img/3.png)

Commands run:

```
set session:user123 "data"   -> OK
get session:user123          -> "data"
set otherkey "oops"          -> (error) NOPERM No permissions to access a key
```

![alt text](img/4.png)

The `NOPERM` error on `set otherkey` confirms that `app_user` is correctly restricted to `session:*` keys only.

#### monitoring User Test

```bash
redis-cli -u redis://monitoring:monitorPwd@127.0.0.1:6379
```

![alt text](img/5.png)

Commands run:

```
INFO server              -> (full server information returned)
set testkey "attempt"    -> (error) NOPERM User monitoring has no permissions to run the 'set' command
```

The monitoring user can read server information but is blocked from any write operations, confirming read-only RBAC enforcement.

### 4.4 Step 4: Generate TLS Certificates for Redis

Self-signed certificates were generated using OpenSSL:

![alt text](img/6.png)

```bash
sudo mkdir -p /etc/redis/tls

# CA key and certificate
sudo openssl genrsa -out /etc/redis/tls/ca.key 4096
sudo openssl req -x509 -new -nodes -key /etc/redis/tls/ca.key -sha256 -days 365 \
  -out /etc/redis/tls/ca.crt \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=redis-lab-ca"

# Server key and CSR
sudo openssl genrsa -out /etc/redis/tls/redis.key 4096
sudo openssl req -new -key /etc/redis/tls/redis.key -out /etc/redis/tls/redis.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"

# Sign the server certificate with the CA
sudo openssl x509 -req -in /etc/redis/tls/redis.csr \
  -CA /etc/redis/tls/ca.crt -CAkey /etc/redis/tls/ca.key \
  -CAcreateserial -out /etc/redis/tls/redis.crt -days 365 -sha256
```

![alt text](img/7.png)

Files generated in `/etc/redis/tls/`: `ca.key`, `ca.crt`, `redis.key`, `redis.csr`, `redis.crt`

### 4.5 Step 5: Configure Redis for TLS

The following TLS settings were added to `/etc/redis/redis.conf`:

```
port 0
tls-port 6379
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-auth-clients no
```

File permissions were set so that the Redis service could read the certificates:

```bash
sudo chown redis:redis /etc/redis/tls/ca.crt /etc/redis/tls/redis.crt /etc/redis/tls/redis.key
sudo chmod 640 /etc/redis/tls/ca.crt /etc/redis/tls/redis.crt /etc/redis/tls/redis.key
sudo chmod 644 /etc/redis/tls/ca.crt
```

Redis was restarted successfully:

```bash
sudo systemctl restart redis-server
# Status: active (running)
```

### 4.6 Step 6: Test TLS Connection to Redis

The CA certificate was copied to the home directory to allow the client user to read it:

```bash
sudo cp /etc/redis/tls/ca.crt ~/redis-ca.crt
```

![alt text](img/8.png)

TLS connection test:

```bash
redis-cli --tls --cacert ~/redis-ca.crt \
  -u rediss://app_user:appStrongPwd@127.0.0.1:6379
```

Commands run inside the TLS session:

```
set session:user456 "secure"   -> OK
get session:user456             -> "secure"
```

The `rediss://` scheme (double s) indicates a TLS-secured connection. Both read and write operations functioned correctly over the encrypted connection.

![alt text](img/9.png)

---

## 5. Part B - Securing MongoDB

### 5.1 Step 1: Start MongoDB and Create Root Admin User

MongoDB was initially started without authentication to create the first admin user. The following user was created in the `admin` database:

```javascript
use admin;
db.createUser({
  user: "rootAdmin",
  pwd: "rootStrongPwd",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
});
// Output: { ok: 1 }
```

![alt text](img/10.png)

### 5.2 Step 2: Enable Authentication in mongod.conf

The following was set in `/etc/mongod.conf`:

```yaml
security:
  authorization: enabled
```

MongoDB was restarted to apply the change.

### 5.3 Step 3: Verify Authentication Enforcement

Connecting without credentials:

```bash
mongosh --host 127.0.0.1 --port 27017
```

```javascript
show dbs;
// Output: MongoServerError[Unauthorized]: Command listDatabases requires authentication
```

This confirms that unauthenticated access is blocked.

![alt text](img/11.png)

Connecting with credentials:

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u rootAdmin -p rootStrongPwd \
  --authenticationDatabase admin
```

![alt text](img/12.png)

```javascript
db.runCommand({ connectionStatus: 1 });
```

Output confirmed:

```
authenticatedUsers: [ { user: 'rootAdmin', db: 'admin' } ]
authenticatedUserRoles: [
  { role: 'dbAdminAnyDatabase', db: 'admin' },
  { role: 'readWriteAnyDatabase', db: 'admin' },
  { role: 'userAdminAnyDatabase', db: 'admin' }
]
ok: 1
```

![alt text](img/13.png)

### 5.4 Step 4: Create Application Role and User (RBAC)

Logged in as rootAdmin, a custom role and application user were created in the `myapp` database:

```javascript
use myapp;

db.runCommand({
  createRole: "myAppRole",
  privileges: [
    {
      resource: { db: "myapp", collection: "customers" },
      actions: ["find", "insert", "update", "remove"]
    }
  ],
  roles: []
});
// Output: { ok: 1 }

db.createUser({
  user: "appUser",
  pwd: "appStrongPwd",
  roles: [{ role: "myAppRole", db: "myapp" }]
});
// Output: { ok: 1 }
```

![alt text](img/14.png)

`myAppRole` grants privileges only on the `myapp.customers` collection. No other databases or collections are accessible.

### 5.5 Step 5: Test RBAC for appUser

Connected as appUser:

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u appUser -p appStrongPwd \
  --authenticationDatabase myapp
```

![alt text](img/15.png)

Allowed operations on `myapp.customers`:

```javascript
use myapp;
db.customers.insertOne({ name: "Pema", city: "Phuntsholing" });
// Output: { acknowledged: true, insertedId: ObjectId('...') }

db.customers.find();
// Output: [ { _id: ObjectId('...'), name: 'Pema', city: 'Phuntsholing' } ]
```

![alt text](img/16.png)

Denied operation on `admin` database:

```javascript
use admin;
db.system.users.find();
// Output: MongoServerError[Unauthorized]: not authorized on admin to execute command
```

![alt text](img/17.png)

This demonstrates that appUser is correctly confined to `myapp.customers` only.

### 5.6 Step 6: Generate TLS Certificates for MongoDB

```bash
sudo mkdir -p /etc/mongo/tls

# CA key and certificate
sudo openssl genrsa -out /etc/mongo/tls/ca.key 4096
sudo openssl req -x509 -new -nodes -key /etc/mongo/tls/ca.key -sha256 -days 365 \
  -out /etc/mongo/tls/ca.pem \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=mongo-lab-ca"

# Server key and certificate
sudo openssl genrsa -out /etc/mongo/tls/mongo.key 4096
sudo openssl req -new -key /etc/mongo/tls/mongo.key -out /etc/mongo/tls/mongo.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"
sudo openssl x509 -req -in /etc/mongo/tls/mongo.csr \
  -CA /etc/mongo/tls/ca.pem -CAkey /etc/mongo/tls/ca.key \
  -CAcreateserial -out /etc/mongo/tls/mongo.crt -days 365 -sha256

# Combine key and cert into a single PEM file
sudo bash -c 'cat /etc/mongo/tls/mongo.key /etc/mongo/tls/mongo.crt > /etc/mongo/tls/mongo.pem'
```

![alt text](img/18.png)

Files generated: `ca.key`, `ca.pem`, `mongo.key`, `mongo.csr`, `mongo.crt`, `mongo.pem`

### 5.7 Step 7: Configure MongoDB for TLS

The `net` section in `/etc/mongod.conf` was updated:

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/tls/mongo.pem
    CAFile: /etc/mongo/tls/ca.pem
    allowConnectionsWithoutCertificates: true

security:
  authorization: enabled
```

![alt text](img/19.png)

Permissions were set and MongoDB was restarted:

```bash
sudo chown mongodb:mongodb /etc/mongo/tls/mongo.pem /etc/mongo/tls/ca.pem
sudo chmod 600 /etc/mongo/tls/mongo.pem
sudo chmod 644 /etc/mongo/tls/ca.pem
sudo systemctl restart mongod
# Status: active (running)
```

### 5.8 Step 8: Test TLS Connection to MongoDB

The CA certificate was copied to the home directory:

```bash
sudo cp /etc/mongo/tls/ca.pem ~/mongo-ca.pem
```

TLS connection using `localhost` (matching the CN in the certificate):

```bash
mongosh --host localhost --port 27017 \
  --tls --tlsCAFile ~/mongo-ca.pem \
  -u appUser -p appStrongPwd \
  --authenticationDatabase myapp
```

![alt text](img/20.png)

Commands run inside the TLS session:

```javascript
use myapp;
db.customers.insertOne({ name: "TLS Test", city: "Thimphu" });

db.customers.find();
```

Both insert and find operations succeeded over an encrypted TLS connection.

![alt text](img/21.png)

---

## 6. Part C - Security Audit

### 6.1 Redis Security Audit

| Check | Test Performed | Result |
|-------|---------------|--------|
| Authentication enforced | Connected without credentials, ran commands | Failed as expected - default user is disabled |
| ACL rules for app_user | Set a non-session:* key as app_user | NOPERM error - key access restricted correctly |
| ACL rules for monitoring | Ran set command as monitoring user | NOPERM error - write access blocked correctly |
| monitoring read access | Ran INFO server as monitoring user | Full server info returned correctly |
| Dangerous commands restricted | admin commands not available to app_user or monitoring | Confirmed via NOPERM errors |
| TLS enforced | Connected using rediss:// with CA cert | TLS connection established successfully |
| Read/write over TLS | Set and get session key over TLS | Both operations succeeded |

**What is secure:**
- Default user is disabled, forcing all clients to authenticate.
- app_user is confined to session:* keys only, preventing access to any other data.
- monitoring user has strictly read-only access with no write permissions.
- TLS encrypts all traffic between client and Redis server.

**What can still be improved:**
- Passwords used in the lab (adminStrongPwd, appStrongPwd) are predictable. In production, randomly generated passwords of at least 32 characters should be used.
- `tls-auth-clients` was set to `no` for the lab. In production, client certificates should also be required to prevent unauthorized clients from connecting even if they have the CA cert.
- Redis is bound only to 127.0.0.1, which is correct for a local lab. In production, firewall rules should further restrict access to the Redis port.

### 6.2 MongoDB Security Audit

| Check | Test Performed | Result |
|-------|---------------|--------|
| Authentication enforced | Connected without credentials, ran show dbs | Unauthorized error - access correctly denied |
| rootAdmin roles verified | connectionStatus command | All three admin roles confirmed |
| appUser RBAC - allowed ops | insert and find on myapp.customers | Both operations succeeded |
| appUser RBAC - denied ops | find on admin.system.users | Unauthorized error - correctly blocked |
| TLS enforced | Connected without --tls flag | Connection refused |
| TLS connection with auth | Connected with --tls and CA cert | Connection established successfully |
| Read/write over TLS | insert and find over TLS session | Both operations succeeded |

**What is secure:**
- Authentication is enforced; no unauthenticated client can access any database.
- appUser is restricted to only the `myapp.customers` collection via a custom role, demonstrating the principle of least privilege.
- TLS is set to `requireTLS` mode, meaning no unencrypted connections are accepted.
- MongoDB is bound to 127.0.0.1, limiting exposure to the local machine only.

**What can still be improved:**
- `allowConnectionsWithoutCertificates: true` was set for the lab. In production, this should be set to `false` so that mutual TLS is enforced.
- The self-signed certificates used here are suitable only for a lab environment. In production, certificates should be issued by a trusted Certificate Authority.
- An audit log should be enabled in MongoDB (available in Enterprise edition) to track all authentication attempts and privileged operations.
- Password rotation policies should be enforced for all database users.

---

## 7. Conclusion

This practical demonstrated the implementation of three core security principles - authentication, encryption, and role-based access control - on both Redis and MongoDB. By disabling the default Redis user and configuring ACL rules, it was shown that different users can be given precisely scoped access to commands and key patterns. TLS certificates generated with OpenSSL were used to encrypt traffic between clients and both database servers, protecting data in transit. In MongoDB, enabling `authorization: enabled` and creating a custom role confined the application user to a single collection, preventing any unauthorized access to administrative data. The security audit confirmed that all negative test cases (unauthenticated access, unauthorized key or collection access, non-TLS connections) failed as expected, while all permitted operations succeeded. These security measures are fundamental to protecting NoSQL databases in real-world deployments.

---