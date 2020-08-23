

## 7. Example of secure service certificates


```
$ oc annotate service django-psql-example service.beta.openshift.io/serving-cert-secret-name=django-psql-secret
service/django-psql-example annotated
```

```
$ oc get secret django-psql-secret
NAME                 TYPE                DATA   AGE
django-psql-secret   kubernetes.io/tls   2      75s
```


```
oc get secret django-psql-secret -o jsonpath="{.data['tls\.crt']}" | base64 -d | openssl x509 -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 7909075559043750593 (0x6dc2ae511e86dac1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = openshift-service-serving-signer@1596638040
        Validity
            Not Before: Aug 12 19:10:18 2020 GMT
            Not After : Aug 12 19:10:19 2022 GMT
        Subject: CN = django-psql-example.secure-demo.svc
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
```


```
        volumeMounts:
        - name: django-psql-secret
          readOnly: true
          mountPath: /usr/local/etc/ssl/certs
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 3
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 3
        resources:
          limits:
            memory: 512Mi
      volumes:
      - name: django-psql-secret
        secret:
          secretName: django-psql-secret
```


```
oc adm policy add-scc-to-user anyuid -z default
securitycontextconstraints.security.openshift.io/anyuid added to: ["system:serviceaccount:secure-demo:default"]
```

```
kubectl create deployment --image nginx my-nginx
```



```
oc create configmap nginx-ca --from-literal=key1=foo
```

```
 oc get cm nginx-ca -o yaml --export
Flag --export has been deprecated, This flag is deprecated and will be removed in future.
apiVersion: v1
data:
  key1: foo
kind: ConfigMap
```

```
oc annotate configmap nginx-ca service.beta.openshift.io/inject-cabundle=true
configmap/nginx-ca annotated
```

```
oc get pod --field-selector status.phase=Running
NAME                          READY   STATUS    RESTARTS   AGE
django-psql-example-2-2h7j2   1/1     Running   0          2m56s
postgresql-1-4t8gn            1/1     Running   0          68m
```
