# Installation

1. Copy the files
2. Install containerd

        sudo apt install ./containerd.io_1.7.24-1_amd64.deb          

3. Check the /etc/continerd and create file /etc/containerd/config.toml
4. Run the default configs for containerd and copy them to the new file
        
        containerd config default

copy the output to /etc/containerd/config.toml

5. Change the Sytemcgroup from false to true

                SystemdCgroup = true

6. Change the pause image registry

                sandbox_image = "registry.k8s.io/pause:3.9" #change to the local docker registry <ip-address>:<port>/pause'3.9

7. Restart containerd 
        
        sudo systemctl restart containerd