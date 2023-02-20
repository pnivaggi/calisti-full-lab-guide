
## Enable ACL on kafka

First we need to enable ACL access and default to deny access if no ACL is defined 

```
    authorizer.class.name=kafka.security.authorizer.AclAuthorizer
    allow.everyone.if.no.acl.found=false
```
to 
```
 spec:
   readOnlyConfig: |
```
in the kafkacluster CR, so at the end you will have something like:
```
 spec:
   readOnlyConfig: |
    ...
    authorizer.class.name=kafka.security.authorizer.AclAuthorizer
    allow.everyone.if.no.acl.found=false
```

To edit the CR launch: 

```bash
kubectl edit kafkacluster kafka -n kafka
```

We apply the kafka ACLs in order to allow traffic for the demoapp prodcers/consumers:
```bash
kubectl apply -f $HOME/lab/acl/demoapp_kafka_acls.yaml
```

