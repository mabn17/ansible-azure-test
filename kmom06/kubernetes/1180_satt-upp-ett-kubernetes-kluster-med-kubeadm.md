---
author:
    - moc
category:
    - devops
    - kubernetes
revision:
    "2020-11-27": (A, moc) Skapad inför HT2020.
...

Sätt upp ett kubernetes kluster med kubeadm {#intro}
=======================================================
Kubernetes tillåter oss att automatiskt hantera distributionen och skalning av våra applikationer.
Nästan alla molntjänster som Azure, AWS och GCP har sina egna sätt att hantera *kubernetes cluster*. Det är oftast ganska dyrt och, om man byter plattform behövs det även ändras en hel del i sin kod.

I denna artikeln skall vi se hur man själv kan sätta upp ett eget *self-managed kubernetes cluster* med hjälp av verktyget `kubeadm` som fungerar på alla tjänster och gör det billigare i längden.

<!--more-->


Skapa ett Nätverk med master och woker nodes {#create}
--------------------------------------------------------

Vi behöver sätta upp våran infrastruktur själva då `kubeadm` inte gör detta åt oss. För att göra detta använder jag mig av [`az cli`](https://docs.microsoft.com/sv-se/cli/azure/install-azure-cli) som är azures egna klient (`pip install azure-cli`), ni skall göra detta i ansible men häng gärna med för att se vad som skapas. Innan ni använder `az` kommandon behöver ni också logga in i klienten: `az login`.

Ni kan använda er av samma bas som tidigare men det är två saker som skiljer sig:

1. Använd ett Vitituellt närverk samt Subnet.
2. Master noden skall ha en annan `vm_size`.

Jag börjar med att skapa nätverket.

```bash
$ az network vnet create -g {RESOURCE_GROUP} -n KubeVNet --address-prefix 172.0.0.0/16 --subnet-name MySubnet --subnet-prefix 172.0.0.0/24
```

Nästa steg är nu att skapa en "master node" som kommer att bli våran *host* till klustret vi skall skapa. På riktigt skulle vi velat ha flera master nodes så att man har minst en backup om denna skulle krascha, men det behöver vi inte tänka på.

Denna virtuella maskinen kräver lite mer kraft så därför uppgraderar vi till `Standard_B2s` som har en extra CPU och mer RAM. För att skapa master noden kör vi följande kommando:

```bash
$ az vm create -n kube-master -g {RESOURCE_GROUP} --image Debian --size Standard_B2s --storage-sku Standard_LRS --os-disk-size-gb 30 --ssh-key-value /path/to/ssh_key.pub --public-ip-address-dns-name kubeadm-master
```

Nu kan vi skapa våra workers som är maskinerna vilket kör våra containeriserade applikationer.   
Vi vill ha **två stycken** av dessa då våran applikation inte har så många besökare och det är mest för att testa på, annars är det vanligt att ha ett ojämnt antar noder ca 3-5 stycken per master.   
Anledningen till detta är att om man har ett jämnt antal är det mycket lättare att hamna i en situation där nätverket går sönder och 50% av besökarna hamnar på vardera "sida". Med ett udda antal servrar kan man inte (eller åtminstone lika lätt) ha en situation där mer än en partition i nätverket tycker att den har majoritetskontroll.

Jag skapar nu en worker:

```
$ az vm create -n kube-worker-1 -g {RESOURCE_GROUP} --image Debian --size Standard_B1s --storage-sku Standard_LRS --os-disk-size-gb 30 --ssh-key-value /home/moc/.ssh/id_rsa.pub --public-ip-address-dns-name kubeadm-worker-1
```

Jag repeterar kommandot igen, men byter ut `-1` till `-2`. En rekommendation senare är att skapa en ny [loop](https://docs.ansible.com/ansible/2.9/user_guide/playbooks_loops.html#with-sequence) så att man enkelt kan öka eller minska antalet maskiner genom att ändra en siffra.


Installera paket och andra beroenden {#install}
--------------------------------------------------------
`kubeadm` kräver att alla noder skall ha utvecklingsmiljön installerad för att köras då den inte sätter upp infrastukturen själv.

Vi behöver installera följande på **alla** våra virtuella maskiner:

```bash
$ sudo apt update
$ sudo apt install gnupg software-properties-common docker.io -y
$ sudo systemctl enable docker
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
$ sudo apt update
$ sudo apt install kubeadm -y
```

Först uppdaterar vi `apt`, installerar alla paket och startar docker. Sedan hämtar vi allt som behövs för att både `kubernetes` och `kubeadm` skall fungera.

När allt är installerat hoppar vi tillbaka till **master noden** och förbereder vårat kluster.

Med kommandot `sudo kubeadm init` initierar vi våran master node, när kommandot är klart kommer vi att få en lång output, det vi är intresserade av är de sista raderna som skriver ut ett kommando, mitt ser ut såhär:

```
kubeadm join 10.0.1.4:6443 --token pp0wxs.uzc32db180qdpcnh \
    --discovery-token-ca-cert-hash sha256:56a390a472b58c78a82bbceb535859fdb053a6c94df57331701fe4e3fbe18db8
```

Denna kommer vi att behöva spara undan och sedan köra på våra workers så att de kan koppla sig till våran master. När det är klart vill vi ändra så att vi slipper köra `sudo` när vi skall jobba vidare med våra kubernetes kommandon:

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Innan våran master nod är klar att ta emot workers behöver vi installerat en pakethanterare för kubernetes och "network manager addon" som skall hantera vårat kluster.


Som pakethanterare använder vi verktyget [`helm`](https://helm.sh/), detta kommer främst att underlätta när vi sätter upp våran LoadBalancer senare:

```bash
$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```


Det finns många network addons för `kubeadm` att välja mellan, alla fungerar för det mesta men kräver en förberedelser. Jag väljer att köra på [`weave`](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/) då man bara behöver "köra" filen:

```bash
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

När allt det är nedladdad och färdigt kan vi köra `kubectl get pods -n kube-system` för att kolla så våran miljö är redo. Om allt har status "Running" kan vi börja att koppla våra workers.

```
NAME                                  READY   STATUS    RESTARTS   AGE
coredns-f9fd979d6-6cb8p               1/1     Running   0          2m17s
coredns-f9fd979d6-fspvm               1/1     Running   0          2m17s
etcd-kube-master                      1/1     Running   0          2m18s
kube-apiserver-kube-master            1/1     Running   0          2m18s
kube-controller-manager-kube-master   1/1     Running   0          2m18s
kube-proxy-9xt6k                      1/1     Running   0          2m17s
kube-scheduler-kube-master            1/1     Running   0          2m17s
weave-net-xlh7t                       2/2     Running   0          93s
```


Nu kan vi koppla våra workers till master genom att ssh:a in på dem och köra samma kommando som vi tidigare sparade undan, kör kommandot som en `superuser`:

```
$ sudo kubeadm join 10.0.1.4:6443 --token nzcdja.3w87ie3zzbzd3lf7 \
    --discovery-token-ca-cert-hash sha256:f9f342c029417d04e6cd71a9ddbe8ffc7c12da79a5bd614650def56ae351f74b
```

När det är klart skall ni få ett meddelande liknande:

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

Testa att logga in på eran master nod och kör kommandot `kubectl get nodes` så skall alla kopplade noder listas:

```
moc@kube-master:~$ kubectl get nodes
NAME            STATUS   ROLES    AGE    VERSION
kube-master     Ready    master   11m    v1.19.4
kube-worker-1   Ready    <none>   5m8s   v1.19.4
kube-worker-2   Ready    <none>   74s    v1.19.4
```


Avslutningsvis {#avslutning}
--------------------------------------------------------
Nu är allt uppe och redo att köras, det som återstår är att applicera våran kubernetes kod, automatisera infrastrukturen med ansible och fixa dns zonen. Kör på och lycka till.

Jag kan rekommendera att titta igenom [dokumentationen](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) om ni kör fast.

Är ni mer nyfikna finns det också en gratis video-serie där Scott Lowe sätter upp `kubeadm` på AWS. Den [första videon](https://kube.academy/lessons/bootstrapping-a-cluster-with-kubeadm) är relevant då han endast visar samma process med en annan network manager addon och pratar om verktyget fungerar och vad det gör.

