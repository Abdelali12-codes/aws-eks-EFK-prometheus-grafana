# ebs csi driver and external-dns
1. create trust relationship

```
cat > aws-role-trust-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::080266302756:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/05E86AA4B06296FD400B0E3B017F1679"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-west-2.amazonaws.com/id/05E86AA4B06296FD400B0E3B017F1679:aud": "sts.amazonaws.com",
                    "oidc.eks.us-west-2.amazonaws.com/id/05E86AA4B06296FD400B0E3B017F1679:sub": "system:serviceaccount:default:external-dns"
                }
            }
        }
    ]
}
EOF

aws iam create-role --role-name aws-external-dns-role --assume-role-policy-document file://aws-role-trust-policy.json
```
2. create policy
```
cat > external-dns-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF

aws iam create-policy --policy-name external-dns-policy --policy-document file://external-dns-policy.json
```
3. attach policy to the role
```
aws iam attach-role-policy --role-name aws-external-dns-role --policy-arn "arn:aws:iam::080266302756:policy/external-dns-policy"
```
4. annotate the external-dns serviceaccount
```
kubectl annotate serviceaccount -n default external-dns eks.amazonaws.com/role-arn=arn:aws:iam::080266302756:role/aws-external-dns-role
```
5. create interactive terminal to pod
```
kubectl run --rm -i --tty nginx --image=nginx --restart=Never -- /bin/bash
```
6. wget lunix command 
```
wget url

-q turn off ouput
-O save file under different name 
-P download file to specific directory
-c resuming download 
-b download in backgroud 
-i file.tx file contains all the urls one under one
--ftp-user --ftp-password 

```


