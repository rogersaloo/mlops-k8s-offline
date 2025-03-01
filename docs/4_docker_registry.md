# Installation
    

## Step1: Generate the ssl certificates
1. Create the certificates directory. 
  
        mkdir certs
    
2. Create and Copy below content to **openssl.conf**
    
    *Update **Docker Server IP** with the IP address of your server where you will be running docker registry*
    
3. Generate the certificate and private key. Expires in 1 year

        openssl req -x509 -newkey rsa:4096 -days 365 -config openssl.conf -keyout certs/domain.key -out certs/domain.crt
    
4. Verify the creatted certificates  

        openssl x509 -text -noout -in certs/domain.crt

5. Update the insecure certificates

    After adding the certificates that now live in the local server, update the docker daemon configuration to allow the usage of the private docker registry.

    a. Navigate to `/etc/docker/daemon.json`  update the following configuration on the local server with the registry.

        {
          "insecure-registries": ["192.168.100.148:6000"]
        }

    b. Restart the docker daemon
      
        sudo systemctl restart docker

## Step2: Deploy registry with the TLS certificates (Recommended)
    
2. Set up the docker compose with TLS. The volume retains the images of uploaded to the registry even after updates. 
    
    Run theeeee docker compose 
    
            docker compose up -d
    
    **NOTE**: To check the registry is running on the deployed IP 
            
        curl -v https://<ip address>:<port>/v2/
    
If there is a TLS error then perform step3




### Step3: Error - Update the CA certificates *optional()

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