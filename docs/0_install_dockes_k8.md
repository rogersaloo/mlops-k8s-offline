# Local k8s cluster
The repo describes the installation of kubernates with kubeadm and set up a local cluster that is disrtibuted among multiple woker and control nodes.

Before proceeding there are a number of local images that need to be downloaded, saved and loaded to the local environment [save_load_k8_components](../local-images/save_k8_components.md). The images consist of the k8 components that are used to set up the local cluster.

In this local installation we use the [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) to set up the kubernetes(k8s) cluster. Additionlly, for the cluster network we use the Weave Net for the network.

Running the local kubernetes cluster on an offline environment requires the following;

## Install the docker on the local machine
Install docker on the environment and ensure that it is running successfully. Refer [here](../docker-registry/install_docker.md)


## Add kubernetes components
Add all the tools and the components to the machine that will run one of the master nodes. 

### Online environment
On the online environment carry out the following steps to obtain the kubeadm components
1. Add the kuberenetes signing key
    
    	curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    
2. Add the software repositories 
    
        echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
        
    Update the packages 
    
        sudo apt update
    
3. Install the kubernetes tools
    
        sudo apt install kubelet kubectl kubeadm

    
4. Check the versions to make sure that the installion has been completed successfully
    
        kubeadm version
        kebelet version
        kubectl version
        
### Offline environment
Please refer to the different images and component needed to be downloaded for the offline environment [here](../local-images/save_k8_components.md). 
Make sure that you have.
1. The kubernetes tools `kubeadm`, `kubelet` and `kubectl`. To install the dowloaded tools amd run the package install with. 

        apt install <path/to/downloaded/file.deb>

## Install Kubernetes on environment 
The kubernetes installation is for `version v1.30.6`. Also check out the version of coredns, pause and etcd.

2. Install other cri-tools, ethol and connrack

        tar -zxvf crictl-v1.31.1-linux-amd64.tar.gz
        sudo mv crictl /usr/local/bin/
        sudo dpkg -i ethtool_*.deb conntrack_*.deb
        crictl --version
        ethtool --version
        conntrack --version

### Check ports and disable swap
    
1. Check that the required ports are open inorder for the k8 componets to communicate.
        
        >> nc 127.0.0.1 6443 -v
        # The excpected output is a succeded
        << connection to 127.0.0.1 6443 port [tcp/*] succeeded!
    If you recieve a `nc: connect to 127.0.0.1 port 6443 (tcp) failed: Connection refused` make sure that the kubernetes components are well installed. Since kubernetes API sever normally listens on port 6443.


2. The default behavior of a kubelet is to fail to start if swap memory is detected on a node. This means that swap should either be disabled or tolerated by kubelet. Disable all the swaps 
        
        sudo swapoff -a
        ### error
        If output is `swap command not found` use the `sudo swap -a`
        
    Then use the [sed command](https://phoenixnap.com/kb/linux-sed) below to make the necessary adjustments to the */etc/fstab* file that coments out the wap file:
        
        sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
        
### Load containerd modules 
1.  Load the required **containerd** modules. Start by opening the containerd configuration file in 

        sudo nano /etc/modules-load.d/containerd.conf
        
    Add the following two lines to the file:
        
        overlay
        br_netfilter
        
    Save (Press ctl+o then ctrl+x) the file and exit.
        
2. Next, use the [modprobe command](https://phoenixnap.com/kb/modprobe-command) to add the modules into the libux kernel:

        sudo modprobe overlay
        sudo modprobe br_netfilter

    The `br_netfilter` module is essential for maintaining network consistency and reliability in Kubernetes environments, particularly for pod-to-pod communication and network policy enforcement.

    The `overlay` module is used to implement overlay filesystems, which are commonly used by container runtimes like Docker.
        
### Configure Kubernetes networking
1. Open the **kubernetes.conf** file to configure Kubernetes networking:
        
        sudo nano /etc/sysctl.d/kubernetes.conf
        
    Add the following lines to the file:
        
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        net.ipv4.ip_forward = 1
        
    Save the file and exit.
        
2. Reload the configuration by typing:
        
        sudo sysctl --system
        
    
## Set up the cluster
    
Setup the unique name for each of the clusters. If the clusters have the name then there is no need to reset the names. 
    
1. Decide which server will be the master node (in our case the file server and hgx will be masternodes). Then, enter the command on that node to name it accordingly:
        
        sudo hostnamectl set-hostname controlplane
        
    
2. Next, set the hostname on the first worker node `DGX 1` by entering the following command:
    
        sudo hostnamectl set-hostname node01
    
    For the additional DGX 2 and 3 use the  worker nodes, use this process to set a unique hostname on each.
        
        sudo hostnamectl set-hostname node02
        sudo hostnamectl set-hostname node03

    If it is decided to use only the file server as the master node then set up the HGX as `node04`. 
    
    
3. Edit the hosts file on each node by adding the IP addresses and hostnames of the servers that will be part of the cluster.
        
        nano /etc/hosts
        
    Add in the following way. Use the actual IPs by running `hostname -I on each DGX, HGX and file server terminal.

        #<ip-address-of-dgx> allocated name
        192.168.100.12       controlplane    # file server
        192.168.100.22       node01          # DGX 1
        192.168.100.23       node02          # DGX 2
        192.168.100.24       node03          # DGX 3
        192.168.100.25       node04          # HGX (optional if master is 1)

4. Restart the terminal application to apply the hostname change. 

**NOTE: Make sure that all instructions until here a run on every machine that will be part of the cluster.**
    
## Initialize Kubernetes on Master Node
### Configure a cgroup driver

Once you finish setting up hostnames on all the cluster nodes, switch to the master node (`fileserver`) and follow the steps to initialize Kubernetes:

1. Confirm the type of init systemem used by the `fileserver`. We excpect a systemd output.
        
        ps -p 1 
        # The expected output should be `systemd`
          
2. Open the **kubelet** file in a text editor.
        
        sudo nano /etc/default/kubelet
    
    Both the container runtime and the kubelet have a property called "cgroup driver", which is important for the management of cgroups on Linux machines. Add the following line to the file:
        
    
        KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
        
    Save and exit.
        
3.  Reload the configuration and restart the kubelet:
        
        sudo systemctl daemon-reload && sudo systemctl restart kubelet


### Set docker cgroup driver to systemd
1. Open the Docker daemon configuration file:

        sudo nano /etc/docker/daemon.json
        
    Append the following configuration block:
        
        {
                "exec-opts": ["native.cgroupdriver=systemd"
                ],
                "insecure-registries": [
                        "192.168.100.132:5000"
                ],
                "log-driver": "json-file",
                "log-opts": {
                "max-size": "100m"
                },
                "runtimes": {
                        "nvidia": {
                                "args": [],
                                "path": "nvidia-container-runtime"
                        }
                },
               "storage-driver": "overlay2"
        }
        
    Save the file and exit.
    
    **explanation**
    
    - 1-2 The systemd cgroup driver.
    - 4-7 The registry IP address and the port that the private docker registry is located.
    - 8-10 The information for the deamon logs, json format and the maximum number of logs. 
    - 11-16 The container runtime usedin the cluster.
    - 17-18 The storage driver that enbales storage allocation of containers and pod. 
        
2. Reload the configuration and restart Docker:
        
        sudo systemctl daemon-reload && sudo systemctl restart docker
        
### Update the kubeadm configuration file
1. Open the **kubeadm** configuration file:
        
        sudo nano /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

    **error** If the above path does not exist use

         sudo nano /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
        
        
        
    Add the following line to the file:
        
        Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"
        Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --fail-swap-on=false"
        
    Save the file and exit.
    
    ![image.png](assets/kubeadm_config.png)
        
2. Reload the configuration and restart the kubelet:
        
        sudo systemctl daemon-reload && sudo systemctl restart kubelet
        
### Intialize the cluster        
Finally, initialize the cluster by typing:

**NOTE: Run only on the master node (controlplane)
        
    sudo kubeadm init --control-plane-endpoint=controlplane --upload-certs

You should see a similar output


![image.png](assets/ok-kube-init.png)
        
### Troubleshooting error
        
If the following error occurs

![image.png](assets/error-kube-init.png)
        
**Solution 1**
        
1. Navigate to the config toml for containerd
       
        sudo nano /etc/containerd/config.toml
        
2. Change the `disabled_plugins` to `enabled_plugins` 
        
        - disabled_plugins = ["cri"]
        + enabled_plugins = ["cri"]
        
3. Restart the `containerd` service
        
        sudo systemctl restart containerd
        
    
**Solution 2**
        
1. Navigate to the config toml for containerd
        
        sudo nano /etc/containerd/config.toml
        
2. Delete everything and replace it with the following configurationsudo  
        
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
            [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
        
3. Restart the `containerd` service
        
        sudo systemctl restart containerd
            
        
If the above passes without touble shooting you should bea bale to obtain the passed `kube init` output and in figure above.

Once the operation finishes, the output displays a **`kubeadm join`** command at the bottom. Make a note of this command, as you will use it to join the worker nodes to the cluster.
        

        
## Create kube config directory
K8s configurations need to be stored in directory on the master node (file server).
        
1. Create a directory for the Kubernetes cluster:
        
        mkdir -p $HOME/.kube
        
2. Copy the configuration file to the directory:
        
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        
    
3. Change the ownership of the directory to the current user and group using the chown command.
    
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

## Join worker nodes and master node
For the other worker nodes, join the initialized controlplane node by typing in the output. If error occurs refer to [](#troubleshooting-error).  

**This should be runin all the worker nodes. `DGX1`, `DGX2` and the HGX**
        
        
    sudo kubeadm join controlplane:6443 --token <token-output> --discovery-token-ca-cert-hash <hash-output> 
    
Similarly, if the `HGX` is to be used as a second control plane for the `file server`. You can join it to the master node. 

**This should be run in only the master nodes that are joinin**

    sudo kubeadm join controlplane:6443 --token <token-output> --discovery-token-ca-cert-hash <hash-output> --control-plane --certificate-key <cert-key>

## Deploy communication network addon
    
Need to deploy the network used for the communication between the nodes and the pods. 
After running the initialization command the addons for the networks will show and navigate to the network. Otherwise consider usage of the current 
        
A pod network is a way to allow communication between different nodes in the cluster. We use WeaveNet for the deployment of the network.
    
Refer to [downloading k8 components to see how to obtain the Weave Net](../local-images/save_k8_components.md) yaml.

**For online environment with internet run** we can simply run 
    
    kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.29/net.yaml
    

Optional inorder to allow the allocation of some of the pods on the `file server` which serves as the control plane. We need to untain the nodeUntaint the node so that is allows for scheduling. 
    

    kubectl taint nodes --all node-role.kubernetes.io/control-plane-


## Install Helm
1. Copy the previously downloaded helm binary ans extract the binary on the offline machine.
    
        tar -zxvf helm-v3.16.3-linux-amd64.tar.gz

2. Move the helm executable to a directory in the system’s PATH.

        sudo mv linux-amd64/helm /usr/local/bin/helm

3. Verify the Installation: Run `helm version` to confirm that Helm is installed correctly.

**For online environment with internet run**

        curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
        sudo apt-get install apt-transport-https --yes
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
        sudo apt-get update
        sudo apt-get install helm

## Install the Nvidia GPU operator
The nvidia gpu operator is needed in order to allow kubernetes to orchestrate the GPUs in the servers. 

**NOTE: Install the gpu operator on all the worker nodes to be able to use the GPUs in the servers**

1. Copy the downloaded helm from the usb disk

        cp -r /path/to/usb-drive/helm-charts /offline-helm-repo

2. Serve the repository locally on an offline environment. T

        cd /offline-helm-repo/helm-charts
        python3 -m http.server 8080

3. Configure helm to use local repository

        helm repo add offline-repo http://localhost:8080
        helm repo update

4. Install the nvidia gpu operator helm chart

        helm install my-release offline-repo/<chart-name>


**For online environment with internet run**

        helm repo add nvidia https://nvidia.github.io/gpu-operator && helm repo update



[PREV << 3.1 Overview project architecture](architecture.md)    |  3.2 |      [NEXT >> 3.3 - Cluster Monitoring](monitoring_k8s.md)


## Hotfix
1. Remove the pause 3.8 from the local registry
    
        docker rmi registry.k8s.io/pause:3.8
    
    
2. Create local registry follow provided instructions
    - Load the registry2 image
    - Run the docker to start the registry 2 image
        
                sudo docker run -d -p 6000:5000 --restart=always --name registry  registry:2
        

1. Export the file server ip 
    
        `export IP_ADDRESS=192.168.100.148:6000`
    
2. Tag the pause image 
    
    
 
        sudo docker tag registry.k8s.io/kube-apiserver:v1.30.7  $IP_ADDRESS/kube-apiserver:v1.30.7 
        sudo docker tag registry.k8s.io/kube-controller-manager:v1.30.7 $IP_ADDRESS/kube-controller-manager:v1.30.7    
        sudo docker tag registry.k8s.io/kube-scheduler:v1.30.7 $IP_ADDRESS/kube-scheduler:v1.30.7
        sudo docker tag registry.k8s.io/kube-proxy:v1.30.7 $IP_ADDRESS/kube-proxy:v1.30.7
        sudo docker tag registry.k8s.io/etcd:3.5.15-0 $IP_ADDRESS/etcd:3.5.15-0 
        sudo docker tag registry.k8s.io/pause:3.9 $IP_ADDRESS/pause:3.9
        sudo docker tag registry.k8s.io/coredns/coredns:v1.11.3 $IP_ADDRESS/coredns:v1.11.3

    
3. Push to the local registry
    
    
   
        sudo docker push $IP_ADDRESS/kube-apiserver:v1.30.7 
        sudo docker push $IP_ADDRESS/kube-controller-manager:v1.30.7 
        sudo docker push $IP_ADDRESS/kube-scheduler:v1.30.7
        sudo docker push $IP_ADDRESS/kube-proxy:v1.30.7
        sudo docker push $IP_ADDRESS/etcd:3.5.15-0 
        sudo docker push $IP_ADDRESS/pause:3.9
        sudo docker push $IP_ADDRESS/coredns:v1.11.3

    
4. Add the registry to the containerd config
    
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
                [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
                SystemdCgroup = true
        
        [plugins."io.containerd.grpc.v1.cri"]
                sandbox_image = "192.168.100.148:6000/pause:3.9"
    
    
5. Restart containerd
    
        sudo systemctl restart containerd
    
6. Initialize kubeadm
    
        sudo kubeadm init --control-plane-endpoint=controlplane --upload-certs


## Hotfix2 ssl certs

The private docker registry is already running and usable with step1 but accessible on an http network. 
However, the pods (container run on k8s) need and https connection to be able to pull from the registry there in https, 

Also we avoid any abuse of our private registry by setting up ssl certificates. Note that there are other authentication methods for the local registry. 

1. Create the certificates directory. 
  
        mkdir certs
        mkdir docker-registry-data
    
2. Cd to the certs dir and create the file openssl.conf and Copy below content to **openssl.conf**.
    
    *Update **Docker Server IP** with the IP address of your server where you will be running docker registry*
        
        
        [ req ]
        distinguished_name = req_distinguished_name
        x509_extensions     = req_ext
        default_md         = sha256
        prompt             = no
        encrypt_key        = no
        
        [ req_distinguished_name ]
        countryName            = "JP"
        localityName           = "Kyoto"
        organizationName       = "rutilea"
        organizationalUnitName = "rnd"
        commonName             = "<ip address>"
        emailAddress           = "example@rutilea.com"
        
        [ req_ext ]
        subjectAltName = @alt_names
        
        [alt_names]
        IP.1 = 10.100.9.46
      
    
3. Generate the certificate and private key. Expires in 1 year

        openssl req -x509 -newkey rsa:4096 -days 365 -config openssl.conf -keyout certs/domain.key -out certs/domain.crt
    
    
4. Verify the creatted certificates  

        openssl x509 -text -noout -in certs/domain.crt

5. Update the insecure certificates

    After adding the certificates that now live in the local server, update the docker daemon configuration to allow the usage of the private docker registry.

    a. Navigate to `/etc/docker/daemon.json`  update the following configuration on the local server with the registry.

        {
          "insecure-registries": ["10.100.9.46:6000"]
        }

    The final daemon file should look like this
        
        {
                "exec-opts": [
                        "native.cgroupdriver=systemd"
                ],
                "insecure-registries": [
                        "10.100.9.46:6000"
                ],
                "log-driver": "json-file",
                "log-opts": {
                        "max-size": "100m"
                },
                "runtimes": {
                        "nvidia": {
                        "args": [],
                        "path": "nvidia-container-runtime"
                        }
                },
                "storage-driver": "overlay2"
        }

    b. Restart the docker daemon
      
        sudo systemctl restart docker

### Step3: Deploy registry with the TLS certificates (Recommended)
    
2. Set up the docker compose with TLS. The volume retains the images of uploaded to the registry even after updates. 
    
        services:
        
          registry:
            container_name: docker-registry
            restart: always
            image: registry:2
            environment:
              REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
              REGISTRY_HTTP_TLS_KEY: /certs/domain.key
            ports:
              - 6000:5000
            volumes:
              - docker-registry-data:/var/lib/registry
              - ./certs:/certs
        
        volumes:
          docker-registry-data: {}
        
    
    **NOTE**: To check the registry is running on the deployed IP 
            
        curl -v https://<ip address>:<port>/v2/

If the docker compose stil fails due to the overlay you can try a normal docker run
                
                docker run -d \
                --name docker-registry \
                --restart always \
                -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
                -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
                -p 6000:5000 \
                -v docker-registry-data:/var/lib/registry \
                -v $(pwd)/certs:/certs \
                registry:2

    

### Step4: Error - Update the CA certificates

After completeing the above steps and applying a k8 manifest. An error may occur on the untrusted certificates from the SSL to be able to add them to be used by the containerd runtime in the cluster proceed as follows. 

1. Add the registry's CA certificate to the node's trust store: Copy the CA certificate that signed your registry's certificate to each Kubernetes node and add it to the system's certificate store:
    
        sudo cp ca.crt /usr/local/share/ca-certificates/
        sudo update-ca-certificates
    
2. Configure containerd to trust the registry:Edit the containerd configuration file:Add the following configuration:Replace "/path/to/ca.crt" with the actual path to your CA certificate.

    a. Open the containerd configuration file. 
      
          sudo nano /etc/containerd/config.toml
    
    b. Update the file paths to the generated certificates and their
      
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."<ip address>:<port>"]
          endpoint = ["https://<ip address>:<port>"]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."<ip address>:<port>".tls]
          ca_file = "/certs/domain.crt"
      
3. Restart containerd and kubelet:
  
        sudo systemctl restart containerd 
        sudo systemctl restart kubelet

## docker compose problem prune
Run the above compose for the ssl certs before running this.
If the above certificates does not run then continue with the below.

**NOTE: Please backup the DB folder somewhere else first**

        docker system prune --all
        docker volume prune
        docker-compose up