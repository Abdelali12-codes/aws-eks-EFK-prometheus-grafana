# appmesh controller

1. install jq dependency
```
sudo yum install jq -y
```
2. install the pre-release checker
```
curl -o pre_upgrade_check.sh https://raw.githubusercontent.com/aws/eks-charts/master/stable/appmesh-controller/upgrade/pre_upgrade_check.sh
sh ./pre_upgrade_check.sh
```
* if the above command resturn you this message "Your cluster is ready for upgrade. Please proceed to the installation instructions"
* then everything work properly 
3. install the helm chart to install the app mesh controller
* the appmesh controller install a hook that inject two containers in the pods that are labled with the name you specify for them
```
helm repo add eks https://aws.github.io/eks-charts
```
* appmesh custom resource definitions 
```
kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"
```
* create namespace for the appmesh controller
```
kubectl create ns appmesh-system
```
* create service account for the aws appmesh controller
```
cat > aws-appmesh-controller.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::080266302756:role/aws-appmesh-controller
  name: appmesh-controller
  namespace: appmesh-system
EOF
kubectl apply -f aws-appmesh-controller.yaml
```
* if the above command does not run succefully, then make sure that there is not serviceaacount with(appmesh-controller) otherwise remove it and rerun the above command

* the prerequisite of the below step is to have an identity provider for your aws eks cluster otherwise create one
* create identity provider if you do not have it
```
eksctl utils associate-iam-oidc-provider \
    --region=$AWS_REGION \
    --cluster $CLUSTER_NAME \
    --approve
```
* create trust policy role 
```
cat > aws-appmesh-role.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::080266302756:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/D36BC86F32A5B1DC82D12AB3655C103F"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-west-2.amazonaws.com/id/D36BC86F32A5B1DC82D12AB3655C103F:aud": "sts.amazonaws.com",
                    "oidc.eks.us-west-2.amazonaws.com/id/D36BC86F32A5B1DC82D12AB3655C103F:sub": "system:serviceaccount:appmesh-system:appmesh-controller"
                }
            }
        }
    ]
}
EOF

aws iam create-role --role-name aws-appmesh-controller --assume-role-policy-document file://aws-appmesh-role.json
```
* attach policy to the above role
```
aws iam attach-role-policy --role-name aws-appmesh-controller --policy-arn "arn:aws:iam::aws:policy/AWSAppMeshFullAccess"
aws iam attach-role-policy --role-name aws-appmesh-controller --policy-arn "arn:aws:iam::aws:policy/AWSCloudMapFullAccess"
```
* install the appmesh controller helm chart
```
helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=us-west-2 \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller \
    --set tracing.enabled=true \
    --set tracing.provider=x-ray 
```
# References

* https://docs.aws.amazon.com/app-mesh/latest/userguide/getting-started-kubernetes.html

# create serviceaccount for the deployment so the envoy proxy will have permission to pull the virtualNode 

1. create trust policy 
```
cat > aws-workload-role.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::080266302756:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/D36BC86F32A5B1DC82D12AB3655C103F"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-west-2.amazonaws.com/id/D36BC86F32A5B1DC82D12AB3655C103F:aud": "sts.amazonaws.com",
                    "oidc.eks.us-west-2.amazonaws.com/id/D36BC86F32A5B1DC82D12AB3655C103F:sub": "system:serviceaccount:appmesh-ns:appmesh-sa"
                }
            }
        }
    ]
}
EOF
aws iam create-role --role-name aws-workload-role --assume-role-policy-document file://aws-workload-role.json
```
2. create policy that contains the permission to pull the virtualNode
```
cat > aws-policy-sa.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "appmesh:StreamAggregatedResources",
            "Resource": [
                "arn:aws:appmesh:us-west-2:080266302756:mesh/appmesh/virtualNode/php_appmesh-ns",
                "arn:aws:appmesh:us-west-2:080266302756:mesh/appmesh/virtualNode/nginx_appmesh-ns"
            ]
        }
    ]
}
EOF
aws iam create-policy --policy-name aws-policy-sa --policy-document file://aws-policy-sa.json
```
3. attach policy to role 
```
aws iam attach-role-policy --role-name aws-workload-role --policy-arn "arn:aws:iam::080266302756:policy/aws-policy-sa"
```
4. create service account 
```
cat > aws-appmesh-sa.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::080266302756:role/aws-workload-role
  namespace: appmesh-ns
  name: appmesh-sa
EOF
kubectl apply -f aws-appmesh-sa.yaml
```
