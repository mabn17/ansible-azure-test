az network vnet create -g BTH-kmom01-aar-e20886 -n KubeVNet --address-prefix 172.0.0.0/16 --subnet-name MySubnet --subnet-prefix 172.0.0.0/24


az vm create -n kube-master -g BTH-kmom01-aar-e20886 --image Debian --size Standard_B2s --storage-sku Standard_LRS --os-disk-size-gb 30 --ssh-key-value /home/moc/.ssh/id_rsa.pub --public-ip-address-dns-name kubeadm-master


az vm create -n kube-worker-2 -g BTH-kmom01-aar-e20886 --image Debian --size Standard_B1s --storage-sku Standard_LRS --os-disk-size-gb 30 --ssh-key-value /home/moc/.ssh/id_rsa.pub --public-ip-address-dns-name kubeadm-worker-2


sudo apt update
sudo apt install gnupg software-properties-common docker.io -y
sudo systemctl enable docker
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt update
sudo apt install kubeadm -y



sudo kubeadm init

sudo kubeadm join 10.0.1.4:6443 --token 19csxk.yfj5r0lva9atq2ar \
    --discovery-token-ca-cert-hash sha256:b54127d20c5d05f734356470b1fa566abce49e0c65749b1dc2328309823b8a5a 


mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"




rsync -av microblog/kmom06 moc@13.69.184.156:/home/moc/


kubectl apply -f mysql-secrets.yml
kubectl apply -f mysql-pv.yml
kubectl apply -f mysql-deployment.yml
kubectl run -it --rm --image=mysql:5.7 --restart=Never mysql-client -- mysql -h mysql -ppass

kubectl apply -f microblog-deployment.yml

helm install nginx stable/nginx-ingress --set rbac.create=true
kubectl get svc

kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml

kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io

helm repo update

helm install cert-manager --namespace cert-manager --version v0.12.0

kubectl get pods --namespace cert-manager -w

kubectl apply -f issuer-staging.yml
kubectl apply -f issuer-prod.yml


kubectl apply -f ingress.yml