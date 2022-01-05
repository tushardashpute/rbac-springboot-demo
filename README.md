# rbac-springboot-demo

[root@ip-172-31-28-184 rbac-springboot-demo]# kubectl apply -f springboot-deployment.yaml 
deployment.apps/springboot created
[root@ip-172-31-28-184 rbac-springboot-demo]# kubectl apply -f springboot-service.yaml 
service/springboot created
[root@ip-172-31-28-184 rbac-springboot-demo]# alias k=kubectl
[root@ip-172-31-28-184 rbac-springboot-demo]# k get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/springboot-db6684d7b-c8b4f   1/1     Running   0          23s
pod/springboot-db6684d7b-vz2pz   1/1     Running   0          23s
pod/springboot-db6684d7b-wd58t   1/1     Running   0          23s

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)           AGE
service/kubernetes   ClusterIP      10.100.0.1       <none>                                                                   443/TCP           55m
service/springboot   LoadBalancer   10.100.156.235   a3ec204441a054cbd90a56cb3a19dbe9-434268644.us-east-2.elb.amazonaws.com   33333:31170/TCP   12s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/springboot   3/3     3            3           23s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/springboot-db6684d7b   3         3         3       23s
 
Admin will have only view only permissions, he can't patch the deployments.  
  
aws iam create-user --user-name admin
aws iam create-access-key --user-name admin | tee /tmp/create_output.json

cat << EoF > admin_creds.sh

  export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
  export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
  EoF
  
Deploy role can do all operations, he can patch the deployments.  
  
aws iam create-user --user-name deploy
aws iam create-access-key --user-name deploy | tee /tmp/create_output.json

cat << EoF > deploy_creds.sh
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
EoF
  
# kubectl get configmap -n kube-system aws-auth -o yaml | egrep -v "creationTimestamp|resourceVersion|selfLink|uid" | sed '/^ annotations:/,+2 d' > aws-auth.yaml
  
Add the user details to aws-auth.yaml and apply it to k8s cluster.
  
cat aws-auth.yaml 
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::556952635478:role/eksctl-eksdemo-nodegroup-eksdemo-NodeInstanceRole-AGRI5C4SXGJC
      username: system:node:{{EC2PrivateDNSName}}
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
     - userarn: arn:aws:iam::556952635478:user/admin
       username: admin
     - userarn: arn:aws:iam::556952635478:user/deploy
       username: deploy

# . admin_creds.sh 
[root@ip-172-31-28-184 rbac-springboot-demo]# aws sts get-caller-identity 
{
    "Account": "556952635478", 
    "UserId": "AIDAYDLHWZBLM4EAWSZZK", 
    "Arn": "arn:aws:iam::556952635478:user/admin"
}
[root@ip-172-31-28-184 rbac-springboot-demo]# 
[root@ip-172-31-28-184 rbac-springboot-demo]# 
[root@ip-172-31-28-184 rbac-springboot-demo]# k get deploy
Error from server (Forbidden): deployments.apps is forbidden: User "admin" cannot list resource "deployments" in API group "apps" in the namespace "default"
[root@ip-172-31-28-184 rbac-springboot-demo]# 
[root@ip-172-31-28-184 rbac-springboot-demo]# cat admin_creds.sh 
export AWS_SECRET_ACCESS_KEY=BbO+AMouYaWAPenxwwyfChxBCLr1fefEwayoUs+z
export AWS_ACCESS_KEY_ID=AKIAYDLHWZBLBIP5XRHK
[root@ip-172-31-28-184 rbac-springboot-demo]# unset AWS_SECRET_ACCESS_KEY 
[root@ip-172-31-28-184 rbac-springboot-demo]# unset AWS_ACCESS_KEY_ID 
[root@ip-172-31-28-184 rbac-springboot-demo]# k apply -f rbacuser-role-admin.yaml 
role.rbac.authorization.k8s.io/admin created
[root@ip-172-31-28-184 rbac-springboot-demo]# k apply -f rbacuser-role-deploy.yaml 
role.rbac.authorization.k8s.io/deployer created
[root@ip-172-31-28-184 rbac-springboot-demo]# k apply -f rbacuser-role-binding-admin.yaml 
rolebinding.rbac.authorization.k8s.io/read-pods created
[root@ip-172-31-28-184 rbac-springboot-demo]# k apply -f rbacuser-role-binding-deploy.yaml 
rolebinding.rbac.authorization.k8s.io/patch-pods created
[root@ip-172-31-28-184 rbac-springboot-demo]# 

  
 # . deploy_creds.sh  
 [root@ip-172-31-28-184 rbac-springboot-demo]# kubectl scale --replicas=2 deployment/springboot
deployment.apps/springboot scaled
  
  # . admin_creds.sh 
[root@ip-172-31-28-184 rbac-springboot-demo]# kubectl scale --replicas=2 deployment/springboot
Error from server (Forbidden): deployments.apps "springboot" is forbidden: User "admin" cannot patch resource "deployments/scale" in API group "apps" in the namespace "default"

  
  [root@ip-172-31-28-184 rbac-springboot-demo]# k get rs
Error from server (Forbidden): replicasets.apps is forbidden: User "admin" cannot list resource "replicasets" in API group "apps" in the namespace "default"
[root@ip-172-31-28-184 rbac-springboot-demo]# . deploy_creds.sh 
[root@ip-172-31-28-184 rbac-springboot-demo]# k get rs
NAME                   DESIRED   CURRENT   READY   AGE
springboot-db6684d7b   2         2         2       60m
