Make sure that aws cli and helm are installed.

# install EKS
https://eksctl.io/installation/

```
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
```

# create a EKS cluster

`eksctl create cluster` will create a new default cluster


# install prometheus using kubectl and helm

https://docs.aws.amazon.com/eks/latest/userguide/deploy-prometheus.html

In this link, the `helm upgrade` is assigning a gp2 EBS for the nodes in the cluster. After this step, by running `kubectl get all -n prometheus`, should see specially pod/prometheus-server is running.

If not, that means the ebs persistant volume is not attached to the server. THIS IS MOST LIKELY an PERMISSION ISSUE.

Update the EKS policy with 
`
{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::147997160882:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/141D06EB4C954A532420E1C67E94306C"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.us-east-1.amazonaws.com/id/141D06EB4C954A532420E1C67E94306C:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
      }
    }
  }  
`


Then run:

`oidc_id=$(aws eks describe-cluster --name exciting-sculpture-1745069396 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id`


`eksctl utils associate-iam-oidc-provider --cluster exciting-sculpture-1745069396 --approve
`

`eksctl create iamserviceaccount \
        --name ebs-csi-controller-sa \
        --namespace kube-system \
        --cluster exciting-sculpture-1745069396 \
        --role-name AmazonEKS_EBS_CSI_DriverRole \
        --role-only \
        --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
        --approve
`

https://docs.aws.amazon.com/eks/latest/userguide/deploy-prometheus.html

and 

https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html

`aws eks describe-addon-versions --addon-name aws-ebs-csi-driver`

Now the server and alertmanager should be running.

# Run Prometheus

`prometheus-server.prometheus.svc.cluster.local`

`export POD_NAME=$(kubectl get pods --namespace prometheus -l "app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")`

`kubectl --namespace prometheus port-forward $POD_NAME 9093`

Then open `localhost:9093` in local web browser. 

# Next: install Grafana