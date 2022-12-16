# Kubernetes Install (3Node=1Master+2Worker) With Kubeadm on Ubuntu20LTS


~~~
NAME                STATUS   ROLES                  AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
kubernetes-master   Ready    control-plane,master   122m   v1.23.3   10.10.10.113  <none>        Ubuntu 20.04.3 LTS   5.4.0-97-generic   containerd://1.5.5
kubernetes-node01   Ready    <none>                 115m   v1.23.3   10.10.10.114  <none>        Ubuntu 20.04.3 LTS   5.4.0-97-generic   containerd://1.5.5
kubernetes-node02   Ready    <none>                 115m   v1.23.3   10.10.10.115  <none>        Ubuntu 20.04.3 LTS   5.4.0-97-generic   containerd://1.5.5
~~~

**1-** **Tüm Node**'ler de çalıştırılmalıdır.
~~~
sudo vim /etc/hosts

10.10.10.113 kubernetes-master
10.10.10.114 kubernetes-node01
10.10.10.115 kubernetes-node02
~~~

Tüm hostlarda çalıştırılır, $USER aktif kullanıcı ile değiştirilir.
~~~
sudo echo "$USER ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/$USER
~~~

Özellikle Master/s makinelerinde ssh key dağıtılmalıdır ama tüm makineler arası çalıştırılmasında fayda var.
~~~
ssh-keygen

ssh-copy-id $USER@10.10.10.113
ssh-copy-id $USER@10.10.10.114
ssh-copy-id $USER@10.10.10.115
~~~

Gerekli paketleri kuralım.
~~~
sudo apt update && sudo apt -y install curl apt-transport-https vim git wget gnupg2 software-properties-common ca-certificates
~~~

Akabinde **swap** alanını kapatalım ama her makine **reboot** olduğunda bu işlem gerekli, en geçerli yöntem **swap**'sız makine kurulumudur.
~~~
sudo swapoff -a
sudo sed -i '/swap/ s/^\(.*\)$/#\1/g' /etc/fstab
~~~

Şimdi **container runtime** kurulumu yapacağız. Alternatifler **containerd**, **docker** ve **cri-o**'dur fakat **kubernetes dockershim deprecation** yaptığı için biz **containerd** yapacağız, zaten **default** olarak **containerd** geliyor.
~~~
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
~~~
~~~
sudo modprobe overlay
sudo modprobe br_netfilter
~~~
~~~
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
~~~
~~~
sudo sysctl --system

sudo apt install -y containerd

sudo mkdir -p /etc/containerd

sudo containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl restart containerd.service
~~~

Bu arada **container runtime Unix domain socket** yolları aşağıdaki gibidir.
~~~
Docker      > /var/run/docker.sock
Containerd  > /run/containerd/containerd.sock
Cri-o       > /var/run/crio/crio.sock
~~~

Şimdi ise **kubernetes** paketlerini kuralım.
~~~
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt -y install kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

kubectl version --client && kubeadm version
~~~

**Kubectl autocompletion enable** yapalım.
~~~
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
~~~

**2- Sadece Master Node**'sinde çalıştırılmalıdır.

**Optional** olarak aşağıdaki komutla **cluster** için gereken **imajları download** edebilirsiniz fakat yapmazsanızda kurulum esnasında uygulama yapacaktır.
~~~
sudo kubeadm config images pull
~~~

Bu aşamada **POD Network**'ünü de oluşturacağız. Alternatifler **Calico**, **Canal**, **Flannel**, **Romana** ve **Weave**'dir. Biz **Calico** kurulumu yapacağız.
**Kubeadm init** ile **Cluster**'ı oluştururken bunu(**--pod-network-cidr**) dikkate alacağız.

~~~
#calico.yaml dosyasındaki ip bilgisine uygun olarak, bir pod network range'i belirleyerek kubernetes cluster'ımızı oluşturabiliriz,
#subneti "vi calico.yaml" ile değiştirmek istersek, aşağıdaki komutta da ilgili subneti belirtmeliyiz.

curl https://docs.projectcalico.org/manifests/calico.yaml -O

sudo kubeadm init --pod-network-cidr=192.168.0.0/16
~~~

Çıktı aşağıdaki gibi olmalıdır. Hem aşağıdaki adımları takip edeceğiz hem de **kubeadm join** ile **cluster**'a yeni **node**'ler ekleyeceğiz.
~~~
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.10.10.113:6443 --token 6sdzxn.pvgby7g42xq59mfb \
	--discovery-token-ca-cert-hash sha256:c4de5f8322be732828565761ed82274d31f282591017b64ef70891f00bc838a8
~~~

Hemen yapalım.
~~~
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

#export KUBECONFIG=/etc/kubernetes/admin.conf olarak direk çalıştırmıyorum çünkü permission hatası alırım,
export KUBECONFIG=.kube/config
~~~
~~~
sudo systemctl status kubelet.service
~~~

Hemen **calico**'yu da **deploy** edelim. Dosyayı yukarda indirmiştik ve **cluster**'ı ona göre **init** etmiştik.
~~~
kubectl apply -f calico.yaml
~~~

**Kubeconfig** dosyasinin bulundugu dizin, **manifestleri**, **api-server** ve **etcd**'nin manifestlerini incelemek isterseniz,
~~~
ls /etc/kubernetes
ls /etc/kubernetes/manifests
sudo more /etc/kubernetes/manifests/etcd.yaml
sudo more /etc/kubernetes/manifests/kube-apiserver.yaml
~~~

**Cluster**'ı gözlemleyebilirsiniz.
~~~
kubectl cluster-info

kubectl get pods -A [--watch]

kubectl get nodes -o wide

#Master'da init sürecinde token bilgisi verilmisti eğer not almadıysanız aşağıdaki komut ile master node üerinden token'ınızı listeleyebilirsiniz,
kubeadm token list

#Var olan token'ı kaydetmediyseniz, elde etmek için,
kubeadm token create --print-join-command

#Master uzerinden ca cert hash'i elde etmek için,
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
~~~


**3- Sadece Worker None**'sinde çalıştırılmalıdır.

Yukarıda **master node**'yi oluştururken elde ettiğimiz ya da not almadıysak var olan **token**'ı tekrar elde ettiğimiz komut vasıtasıyla **worker node**'leri **cluster**'a dahil edelim.
~~~
#sudo kubeadm join <ip>:6443 --token <token> --discovery-token-ca-cert-hash <ca_cert_hash>

sudo kubeadm join 10.10.10.113:6443 --token 6sdzxn.pvgby7g42xq59mfb \
	--discovery-token-ca-cert-hash sha256:c4de5f8322be732828565761ed82274d31f282591017b64ef70891f00bc838a8
~~~

Tekrar **cluster**'ın durumunu gözlemleyebilirisiniz.

Son olarak **bashrc**'yi **edit**'leyerek aşağıdaki satırları ilgili **node**'lere ekleyelim.
~~~
sudo vi .bashrc
~~~
Tüm Node
~~~
swapoff -a
~~~
Sadece MASTER/S Node
~~~
swapoff -a
export KUBECONFIG=.kube/config
~~~


**NoT** : Ek bilgi

**kubeadm init parametreleri**:<br>
**--control-plane-endpoint** :  set the shared endpoint for all control-plane nodes. Can be DNS/IP<br>
**--pod-network-cidr** : Used to set a Pod network add-on CIDR<br>
**--cri-socket** : Use if have more than one container runtime to set runtime socket path<br>
**--apiserver-advertise-address** : Set advertise address for this particular control-plane node's API server

Örnek olarak **3** ya da daha fazla **Master Node**'yi **Haproxy** ve **Keepalived** aracılığı ile eklemek isterseniz **kubeadm init**'i aşağıdaki gibi çalıştırmalısınız.

Yani **--control-plane-endpoint** adresi belirlediğiniz **vrrp** olacak, **--apiserver-advertise-address** **ip**'si de ilk **master node**'nizin **ip**'sini yazabilirsiniz.
~~~
sudo kubeadm init --control-plane-endpoint="10.10.10.200:6443" --apiserver-advertise-address=10.10.10.113 --pod-network-cidr=192.168.0.0/16 --upload-certs

ya da 

sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --pod-network-cidr=192.168.0.0/16 --upload-certs
~~~

**Init** sürecinde herhangi bir hata almanız halinde aşağıdaki komutu yürüterek **init**‘i tekrar başlatabilirsiniz.
~~~
sudo kubeadm init reset
~~~

Bu vesileyle ortama **Haproxy** ve **Keepalived** dahil ederseniz, **haproxy** için yapabileceğiniz yapılandırmayı da eklemiş olalım.
~~~
frontend kubelets
        bind k8s-fatlan.com:10250
        mode tcp
        default_backend kubelet

frontend kubeproxys
        bind k8s-fatlan.com:10256
        mode tcp
        default_backend kubeproxy

frontend kubeapis
        bind k8s-fatlan.com:6443
        mode tcp
        default_backend kubeapi

backend kubelet
        mode tcp
        option tcplog
        option tcp-check
        balance roundrobin
        server kube-cluster-kubelet-01 10.1.10.52:10250 check port 10250 inter 3000 rise 2 fall 3
        server kube-cluster-kubelet-02 10.1.10.53:10250 check port 10250 inter 3000 rise 2 fall 3
        server kube-cluster-kubelet-03 10.1.10.54:10250 check port 10250 inter 3000 rise 2 fall 3

backend kubeproxy
        mode tcp
        option tcplog
        option tcp-check
        balance roundrobin
        server kube-cluster-kubeproxy-01 10.1.10.52:10256 check port 10256 inter 3000 rise 2 fall 3
        server kube-cluster-kubeproxy-02 10.1.10.53:10256 check port 10256 inter 3000 rise 2 fall 3
        server kube-cluster-kubeproxy-03 10.1.10.54:10256 check port 10256 inter 3000 rise 2 fall 3

backend kubeapi
        mode tcp
        option tcplog
        option tcp-check
        balance roundrobin
        server kube-cluster-kubeapi-01 10.1.10.52:6443 check port 6443 inter 3000 rise 2 fall 3
        server kube-cluster-kubeapi-02 10.1.10.53:6443 check port 6443 inter 3000 rise 2 fall 3
        server kube-cluster-kubeapi-03 10.1.10.54:6443 check port 6443 inter 3000 rise 2 fall 3
~~~
