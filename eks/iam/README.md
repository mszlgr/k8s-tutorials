```
 cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | cut -d'.' -f2 | base64 -d
```
