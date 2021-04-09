### Install kops ###
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops

### Install kubectl ###
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client

### Export variabel yang nanti akan dibutuhkan  ###
export bucket_name=k8s-jokoss-site
export KOPS_CLUSTER_NAME=k8s.jokoss.site
export KOPS_STATE_STORE=s3://${bucket_name}

### Create cluster ###
kops create cluster --zones=ap-southeast-1a \
--node-count=3 \
--master-count=1 \
--node-size=t2.medium \
--master-size=t2.large \
--name=${KOPS_CLUSTER_NAME} \
--ssh-public-key=~/.ssh/id_rsa.pub

kops update cluster --name ${KOPS_CLUSTER_NAME} --yes --admin
kops validate cluster --wait 10m
kubectl get nodes --show-labels

### Install Ingress ###
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.27.1/deploy/static/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.27.1/deploy/static/provider/aws/service-l4.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.27.1/deploy/static/provider/aws/patch-configmap-l4.yaml

git clone https://github.com/kubernetes/ingress-nginx
cd ingress-nginx/deploy/static/provider/aws/
kubectl apply -f deploy.yaml

### Create namespace ###
kubectl create namespace staging
kubectl create namespace production

### Install Certificate Authority ###
snap install helm --classic
ln -s /snap/bin/helm /usr/local/bin/helm
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm --version
helm install cert-manager --namespace cert-manager --version v0.14.3 jetstack/cert-manager
kubectl get pods --namespace cert-manager

### Setting node autoscaler ###
kops edit instancegroups nodes-ap-southeast-1a
# Change this value
  maxSize: 5
  minSize: 1
kops edit cluster
# Add this value on spec
spec:
  additionalPolicies:
    node: |
      [
        {
          "Effect": "Allow",
          "Action": [
            "autoscaling:DescribeAutoScalingGroups",
            "autoscaling:DescribeAutoScalingInstances",
            "autoscaling:SetDesiredCapacity",
            "autoscaling:DescribeLaunchConfigurations",
            "autoscaling:DescribeTags",
            "autoscaling:TerminateInstanceInAutoScalingGroup"
          ],
          "Resource": ["*"]
        }
      ]

wget https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
nano cluster-autoscaler-autodiscover.yaml
# Change this value
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: asia.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.15.6
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --nodes=1:5:nodes-ap-southeast-1a.k8s.jokoss.site
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/k8s.jokoss.site
          env:
            - name: AWS_REGION
              value: ap-southeast-1
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-bundle.crt #/etc/ssl/certs/ca-bundle.crt for Amazon Linux Worker Nodes
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"

### Install EFK ###
cd efk/
kubectl apply -f efk-ns.yaml
cd elastic/
kubectl apply -f p-elasticsearch-svc.yaml
kubectl apply -f p-elasticsearch-statefulset.yaml
kubectl rollout status sts/es-cluster --namespace=kube-logging
kubectl port-forward es-cluster-0 9200:9200 --namespace=kube-logging
cd ../kibana/
kubectl apply -f p-kibana-dpy.yaml
kubectl rollout status deployment/kibana --namespace=kube-logging
kubectl get pods --namespace=kube-logging
kubectl apply -f p-p-ekf-ingress.yaml
kubectl apply -f p-fluentd-dpy.yaml
kubectl get ds --namespace=kube-logging
kubectl get svc --namespace=kube-logging
kubectl get ing --namespace=kube-logging
kubectl apply -f p-counter-pod.yaml

### Install Prometheus Grafana ###
Test1234
