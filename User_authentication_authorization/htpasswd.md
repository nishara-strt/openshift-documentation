# Configuring an HTPasswd Identity Provider in OpenShift

This document explains how to configure an HTPasswd identity provider in OpenShift Container Platform using the `htpasswd` utility and OpenShift OAuth configuration.

Reference Documentation:

* Red Hat OpenShift Documentation: Configuring Identity Providers
* OpenShift Version: 4.8

---

# Prerequisites

Before starting, ensure:

* OpenShift cluster is accessible
* `oc` CLI is installed and configured
* Cluster admin access is available
* `apache2-utils` package is installed

---

# Install htpasswd Utility

On Ubuntu/Debian systems:

```bash
sudo apt install apache2-utils
```

Verify installation:

```bash
htpasswd
```

---

# Create HTPasswd File

Create a working directory:

```bash
mkdir htpasswd
cd htpasswd
```

Create the first user:

```bash
htpasswd -c -B -b users.htpasswd user1 MyPassword!
```

Explanation:

* `-c` → Create a new file
* `-B` → Use bcrypt encryption
* `-b` → Supply password in command line

View the generated file:

```bash
cat users.htpasswd
```

Example Output:

```text
user1:$2y$05$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

# Add Additional Users

Add another user without using `-c`:

```bash
htpasswd -B -b users.htpasswd user2 MyPassword!
```

Verify:

```bash
cat users.htpasswd
```

---

# Create Secret in OpenShift

Create a secret containing the HTPasswd file:

```bash
oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config
```

Verify the secret:

```bash
oc get secret -n openshift-config
```

---

# Configure OAuth Identity Provider

An OAuth Identity Provider in OpenShift Container Platform is the system OpenShift uses to verify a user’s username and password during login. It connects OpenShift to authentication sources like HTPasswd, LDAP, GitHub, or other SSO providers.

Create a file named `htpasswd-cr.yaml`.

Example configuration:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```

Apply the configuration:

```bash
oc apply -f htpasswd-cr.yaml
```

---

# Verify Identity Provider

Check OAuth configuration:

```bash
oc get oauth cluster -o yaml
```

You should see the HTPasswd provider configured under `identityProviders`.

---

# Login Using New User

Login using the newly created user:

```bash
oc login -u user1
```

Verify logged-in user:

```bash
oc whoami
```

Example Output:

```text
user1
```

---

# Grant Cluster Admin Access (Optional)

To provide cluster-admin permissions:

```bash
oc adm policy add-cluster-role-to-user cluster-admin user1
```

Verify:

```bash
oc auth can-i '*' '*'
```

---

# Useful Commands

## List Users

```bash
oc get users
```

## List Identities

```bash
oc get identities
```

## Describe OAuth Configuration

```bash
oc describe oauth cluster
```

---

# Authentication Flow Overview

1. User attempts login using `oc login`
2. OpenShift OAuth server validates credentials
3. OAuth checks the HTPasswd identity provider
4. Credentials are validated against `users.htpasswd`
5. User identity is created in OpenShift
6. RBAC determines permissions

---

# Troubleshooting

## Check OAuth Pods

```bash
oc get pods -n openshift-authentication
```

## View OAuth Logs

```bash
oc logs -n openshift-authentication deployment/oauth-openshift
```

## Validate Secret

```bash
oc describe secret htpass-secret -n openshift-config
```

## Decode Secret

```bash
oc get secret htpass-secret -n openshift-config -o jsonpath='{.data.htpasswd}' | base64 -d
```

---

# Cleanup

Delete the secret:

```bash
oc delete secret htpass-secret -n openshift-config
```

Remove OAuth configuration by editing:

```bash
oc edit oauth cluster
```

Delete the identity provider section.

---

# Summary

In this setup:

* `htpasswd` utility is used to create local users
* The HTPasswd file is stored as a secret
* OpenShift OAuth uses the secret as an identity provider
* Users authenticate through the OAuth server
* RBAC controls authorization and permissions

This is commonly used in:

* Lab environments
* Development clusters
* Testing environments
* Small OpenShift deployments
