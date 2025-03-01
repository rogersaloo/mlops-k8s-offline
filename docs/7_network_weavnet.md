# Installation

1. Load the images

        sudo docker load =i weaveworks-weave-kube.tar
        sudo docker load =i weaveworks-weave-npc.tar

2. Send them to the local registry

3. Edit the image of the 

        image: 'weaveworks/weave-kube:latest' # edit to local registry
        image: 'weaveworks/weave-kube:latest' # Change to local registry

4. Run the yaml on a running cluster of control plane

        sudo kubectl apply weave-daemonset-k8s.yaml


If error we can change image pull policy



