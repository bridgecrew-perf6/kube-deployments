# kube-deployments
deployment yamls for different eks cluster

## PREREQS

* aws credentials/profile for account in question
* aws cli
* kubectl
* eksctl (for configuration alb and k8s integration)

## Caveat

First thing we must do is connect to our k8s cluster
```bash
aws sts get-caller-identity
aws eks update-kubeconfig --name wheeldemo-cluster
```

Then, we must ensure the terraform deployment has succeeded in deploying coredns

```bash
kubectl get deployment coredns -n kube-system
```
If you see something like 

```bash
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   0/2     0            0           5m
```
Then we need to patch the annotations for the deployement, using

```bash
kubectl patch deployment coredns -n kube-system --type json -p='[{"op": "remove", "path": "/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
```
We can then restart the rollout to speed up the process

```bash
kubectl rollout restart deployments -n kube-system
```
## Configure ALB

We need to setup an integration between the aws loadbalancer service and the eks cluster in order for us to leverage k8s ingresses to deploy an alb object. We will leverage an aws tool called eksctl to help make that process easier, since it is a cluster bootstrapping item that will require little maintenance and very few occassions to update (making using terraform for this hard to justify)

First create an oidc provider to allow role assumptions for a k8s service account to an aws iam role

```bash
eksctl utils associate-iam-oidc-provider --cluster wheeldemo-cluster —approve
```
Then we can deploy the clusterrole and rolebinding within k8s

```bash
kubectl apply -f rbac.yaml
```
Then we can create the iam role which will link between the two

```bash
#This leverages a policy created by the previous terraform steps in the wheeldemo repo
eksctl create iamserviceaccount --name alb-ingress-controller --namespace kube-system --cluster wheeldemo-cluster --attach-policy-arn arn:aws:iam::613313135663:policy/ALBIngressControllerIAMPolicy --approve 
```
Then we can apply the ingress controller deployment that will listen to ingress creation events

```bash
 kubectl apply -f alb-ingress-controller.yaml 
```

## Deployment

Now that we've setup the cluster, we get to the easy part, which is deploying the app. Make sure to update the ingress.yaml to select an override security group for the alb. This comes in handy very often, because every docker problem is a network problem, and having the network resources identifiable AND knowing where you can configure them makes troubleshooting so much easier.

```bash
kubectl create namespace demo
kubectl apply -f kube-nginx.yaml 
kubect apply -f ingress.yaml
```
You can check to see how the ingress is going by running

```bash
kubectl get ingress -n demo
```

And you should see something like 

```bash
NAME                 CLASS    HOSTS   ADDRESS                                                                  PORTS   AGE
demo-nginx-ingress   <none>   *       9ddb4511-demo-demonginxing-7b9b-1041056272.us-east-2.elb.amazonaws.com   80      10m
```
Indicating success! Copy the url from the ADDRESS field into a browser and you will be able to see the app, which in this case is an nginx home page 
