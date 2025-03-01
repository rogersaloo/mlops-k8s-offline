1. Install kubeadm

                apt install ./kubernetes-cni_1.4.0-1.1_amd64.deb

                tar -zxvf crictl-v1.31.1-linux-amd64.tar.gz
                sudo mv crictl /usr/local/bin/


                sudo apt install ./cri-tools_1.30.1-1.1_amd64.deb 
                sudo apt install ./ethtool_1_3a5.16-1ubuntu0.1_amd64.deb
                sudo apt install ./conntrack_1_3a1.4.6-2build2_amd64.deb   


        Check the verions
        crictl --version
        ethtool --version
        conntrack --version

        Install kubeadm

                sudo apt install ./kubeadm_1.30.6-1.1_amd64.deb       

2. Install kubectl    
       
                sudo apt install ./kubectl_1.30.6-1.1_amd64.deb

3. Instal kubelet

                sudo apt install ./kubelet_1.30.6-1.1_amd64.deb

4. Check the version of kubeadm 

                kubeadm version
                kubectl version
                kubelet version

5. Go and follow all the procedures in the official documentaion for closed projects


### error *DONT USE
                #sudo dpkg -i ethtool_*.deb conntrack_*.deb
