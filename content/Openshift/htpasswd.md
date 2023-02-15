---
title: "HTPasswd oauth provider"
date: 2023-02-15T13:22:04+01:00
draft: false
---


1. Create htpaswd file
   
```
$ htpasswd -c -B -b htpasswd admin adminpass 
$ htpasswd -c -B -b htpasswd developer devpass
```

2. Create secret
   
```
$ oc create secret generic htpass-secret --from-file=htpasswd -n openshift-config
```

3. Patch oauth cluster
```
$ oc patch oauth/cluster --patch '{"spec":{"identityProviders":[{"name":"htpasswd","mappingMethod":"claim","type":"HTPasswd","htpasswd":{"fileData":{"name":"htpass-secret"}}}]}}' --type=merge
```

4. Give admin user cluster-admin role

```
$ oc adm policy add-cluster-role-to-user cluster-admin admin
```

