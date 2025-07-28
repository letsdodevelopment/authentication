# How to use Authentication in OpenShift

We are using htpasswd as identity. Lets talk on htpasswd.

## htpasswd options availabe

- c to create a file but it must be only used once.
- B Force bcrypt encryption
- PLEASE do not use -b option it means you must entry password in clear text

### Create a file to store password

Below command creates a file for the user ttoc

```bash
htpasswd -B -c samplefilepwd ttoc

# create command add another user, but without using c option, otherwise you will end up overwritten samplefilepwd

htpasswd -B samplefilepwd nimda
cat samplefilepwd

## OutPut ##
ttoc:$2y$05$UgSM6Kqg6QDpgfIpU.PXDuYoktRV9MO0sofAKyLm02YyM10hXM4fK
nimda:$2y$05$CotacmZar8kRZsr8s9/Tb.zpX0RXN29ehZNwrdey/jrJEnXhAN4Ty
```

## Create a secret out of samplefilepwd

This secret must be created into openshift-config project/namespace, and you must have cluster admin role.

```bash
oc whoami

oc create secret generic mysimplepwd --from-file oursecret=samplefilepwd -n openshift-config

# here oursecret is the key, for your secrets stored
```

> Wait for pods in openshift-authentication namespace to restart

### Modify  HTPasswd Identity Provider

```shell
# export the main configuration
oc get oauth cluster -o yaml > oauthcluster.yaml

# take a backup
cp -v oauthconfig.yaml{,.$(date +%F)}
```

#### Edit the below section on oauthcluster.yaml

```shell
oc get oauth cluster -o yaml | grep identityProviders: -A 10

  identityProviders:
  - htpasswd:
      fileData:
        name: localusers # name of the secret
    mappingMethod: claim
    name: myusers # display name
    type: HTPasswd
```

Now once the above section is modified, use `oc replace -f oauthcluster.yaml`

### Assign role to the user

`oc adm policy add-role-to-user cluster-admin nimda`

But user is not visible in the identity or in the users lists

```bash
oc get users
NAME        UID                                    FULL NAME   IDENTITIES
developer   291db44d-6895-46fb-92e8-b80d1912eb5f               myusers:developer
kubeadmin   a690ba2e-d1ed-4632-8278-e224c57159c7               myusers:kubeadmin
nodiesop    9e01d4d2-8525-4b39-afa6-b595ef03503f               myusers:nodiesop
#################################

authentication git:(main) ✗ oc get identity
NAME                IDP NAME   IDP USER NAME   USER NAME   USER UID
myusers:developer   myusers    developer       developer   291db44d-6895-46fb-92e8-b80d1912eb5f
myusers:kubeadmin   myusers    kubeadmin       kubeadmin   a690ba2e-d1ed-4632-8278-e224c57159c7
myusers:nodiesop    myusers    nodiesop        nodiesop    9e01d4d2-8525-4b39-afa6-b595ef03503f
```

Because user must not be present, to provide role to the user. And above user is only visible when he logs in first time.

```shell
~ oc get clusterrolebindings.authorization.openshift.io cluster-admin

NAME            ROLE             USERS   GROUPS           SERVICE ACCOUNTS   USERS
cluster-admin   /cluster-admin           system:masters


➜  ~ oc get clusterrolebindings.authorization.openshift.io cluster-admin-0

NAME              ROLE             USERS      GROUPS   SERVICE ACCOUNTS   USERS
cluster-admin-0   /cluster-admin   nodiesop
```

### Change password for the user

Extract the secret, using the following command. Please I have output that on console using `=-`

```shell
➜  ~ oc extract secret/localusers --to=samplefilepwdv1 -n openshift-config

## OutPut of samplefilepwdv1 i.e. cat samplefilepwdv1
# htpasswd
nodiesop:$2y$05$XAynnhVpdOKsx6vGFcWVB.12yX.dfFtNui8l2Yc5cmtEesbLAuCSm
developer:$2y$05$pqKHT59T3OFQBraOLc3iA.yTxgItt2yW0X537YilQvSK4KDlcqvz6
kubeadmin:$2y$05$v0ePmiLw2SuA1wHlT.UhpePQva3r7PajlqJFgceUqhOYWLHm4cDFm
leader:$2y$05$mA25rqakIZzBFexz6OWQbuxW8TkSbMib39OBQsvRPcyKbbkOHRaFy
user1qa:$2y$05$OqzJczK6pyZ3PSvs0Lo6GupYUCZiVVUtfzikl3k23C1bSQrnezqv6
user1dev:$2y$05$5Qnjn8Y5iF8hc1kw0viKg.PkpG0N.Ut5C5xxB6Z28tOmnLf0a8xUu

```

htpasswd -B samplefilepwdv1 nodieop

you will get prompt to change and confirm the password.

### Update the secret

oc set data secret/localusers --from-file samplefilepwdv1 -n openshift-config

### Wait for pods in openshift-authentication restarts

watch oc get pods -n openshift-authentication

### Confirm login works

oc login -u nodiesop -p secretPasswordYouEnteredEarlier

