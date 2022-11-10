# create aws efs driver 

* create trust policy role 
```
cat > aws-role-trust-policy-efs-driver.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::080266302756:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/BD06364C7FE8173EE0A553D4015E49CA"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-west-2.amazonaws.com/id/BD06364C7FE8173EE0A553D4015E49CA:aud": "sts.amazonaws.com",
                    "oidc.eks.us-west-2.amazonaws.com/id/BD06364C7FE8173EE0A553D4015E49CA:sub": "system:serviceaccount:kube-system:efs-csi-controller-sa"
                }
            }
        }
    ]
}
EOF
aws iam create-role --role-name aws-role-trust-policy-efs-driver --assume-role-policy-document file://aws-role-trust-policy-efs-driver.json
```
* create service account using kubectl
```
cat > efs-csi-controller-sa.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/name: aws-efs-csi-driver
  name: efs-csi-controller-sa
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::080266302756:role/aws-role-trust-policy-efs-driver
EOF
kubectl apply -f efs-csi-controller-sa.yaml
```

* create iam policies
```
cat > efs-csi-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:DescribeAccessPoints",
        "elasticfilesystem:DescribeFileSystems",
        "elasticfilesystem:DescribeMountTargets",
        "ec2:DescribeAvailabilityZones"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:CreateAccessPoint"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/efs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": "elasticfilesystem:DeleteAccessPoint",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
        }
      }
    }
  ]
}
EOF
aws iam create-policy --policy-name aws-efs-csi-driver --policy-document file://efs-csi-policy.json

```
* attach policies to role 
```
aws iam attach-role-policy --role-name aws-role-trust-policy-efs-driver --policy-arn "arn:aws:iam::080266302756:policy/aws-efs-csi-driver"
```
* install the ebs csi helm chart
```
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update

helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
 ```
