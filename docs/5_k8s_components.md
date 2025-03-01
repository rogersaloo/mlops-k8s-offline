## step3 Push the images to local registry
After confirming the local registry is up. Load all the 7 images of kube component.

1. Export the file server ip 
    
    `export IP_ADDRESS=192.168.100.148:6000` # change to the controlpod ip
    
2. Tag the pause image 
    
    
        sudo docker tag registry.k8s.io/kube-apiserver:v1.30.6  $IP_ADDRESS/kube-apiserver:v1.30.6 
        sudo docker tag registry.k8s.io/kube-controller-manager:v1.30.6 $IP_ADDRESS/kube-controller-manager:v1.30.6    
        sudo docker tag registry.k8s.io/kube-scheduler:v1.30.6 $IP_ADDRESS/kube-scheduler:v1.30.6
        sudo docker tag registry.k8s.io/kube-proxy:v1.30.6 $IP_ADDRESS/kube-proxy:v1.30.6
        sudo docker tag registry.k8s.io/etcd:3.5.15-0 $IP_ADDRESS/etcd:3.5.15-0 
        sudo docker tag registry.k8s.io/pause:3.9 $IP_ADDRESS/pause:3.9
        sudo docker tag registry.k8s.io/coredns/coredns:v1.11.3 $IP_ADDRESS/coredns:v1.11.3

3. Check that all the images are loaded locally

        sudo docker image list
    
3. Push to the local registry
    
        sudo docker push $IP_ADDRESS/kube-apiserver:v1.30.6 
        sudo docker push $IP_ADDRESS/kube-controller-manager:v1.30.6 
        sudo docker push $IP_ADDRESS/kube-scheduler:v1.30.6
        sudo docker push $IP_ADDRESS/kube-proxy:v1.30.6
        sudo docker push $IP_ADDRESS/etcd:3.5.15-0 
        sudo docker push $IP_ADDRESS/pause:3.9
        sudo docker push $IP_ADDRESS/coredns:v1.11.3
    
    
4. Add the registry to the containerd config. (You may have already changed it while running containerd)
    
     [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
         [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
         SystemdCgroup = true # change this on the toml
    
     [plugins."io.containerd.grpc.v1.cri"]
        sandbox_image = "192.168.100.148:6000/pause:3.9" #change this on the toml
    
5. Restart containerd

        systemctl restart containerd 
    