## this command is important to create ./kube/config file that will be used by the kubectl

```
aws eks update-kubeconfig --name ekscluster --region us-west-2 --role-arn arn:aws:iam::080266302756:role/EKSClusterStack-Role1ABCC5F0-1BGBXI9PTK3J1 #go to iam console in role tab and search for this keyword EKSClusterStack-MasterRole and 
                                                                                                              copy and paste the role arn
```

# create policy for autoscaler

```
aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document file://autoscaler_policy.json
    
```

2. create iam role for the service account 

```
aws iam create-role --role-name AmazonEKSClusterAutoscalerRole \
        --assume-role-policy-document file://oidc-trust-policy.json \
        --description "eks-cluster-autoscaler"
```

3. attach a policy to the role 

```
aws iam attach-role-policy --role-name AmazonEKSClusterAutoscalerRole --policy-arn=arn:aws:iam::080266302756:policy/AmazonEKSClusterAutoscalerPolicy
```

4. annotate the serviceaccount to use the above role

```
kubectl annotate serviceaccount cluster-autoscaler \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::080266302756:role/AmazonEKSClusterAutoscalerRole
  
```

5. deploy the autoscaler on the cluster

```
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```


# patch the autoscaler pod



```
kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'
  
```

* autoscaler revisions
```
https://github.com/kubernetes/autoscaler/releases
```



#### Pod horizantal autoscaling

1. install the metric server 
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
2. make sure it is running
```
kubectl get deployment metrics-server -n kube-system
```



## the eks cluster dashboard

1. install the eks dashboard
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml
```
2. create eks-admin service account

```
kubectl apply -f dashboard-service-account.yaml
```
3. connect to the cluster using the proxy
```
kubectl proxy
```
4. get the eks-admin secret of the eks-admin service account
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

failed to create service account kube-system/cluster-autoscaler: checking whether namespace "kube-system" exists: Unauthorized