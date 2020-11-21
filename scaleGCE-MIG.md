- [](#)
  - [codelab and github](#codelab-and-github)
  - [git to GCS](#git-to-gcs)
  - [All networking pricing:  static IP, LB etc](#all-networking-pricing--static-ip-lb-etc)
  - [create 2 standalone VM instances](#create-2-standalone-vm-instances)
  - [create MIG:](#create-mig)
  - [create HTTP Load balancer](#create-http-load-balancer)
    - [Create a URL map. The URL map defines which URLs are directed to which backend services.](#create-a-url-map-the-url-map-defines-which-urls-are-directed-to-which-backend-services)
    - [Create the proxy that ties to the created URL map. Create a global forwarding rule that ties a public IP address and port to the proxy.](#create-the-proxy-that-ties-to-the-created-url-map-create-a-global-forwarding-rule-that-ties-a-public-ip-address-and-port-to-the-proxy)

# standalone VM GCE: 

gs:fancy-store-qwiklabsproj/monolith-to-microservices/startup-script.sh
gsutil cp ./monolith-to-microservices/startup-script.sh gs://fancy-store-qwiklabsproj
gsutil -m cp -r monolith-to-microservices gs://fancy-store-qwiklabsproj

## codelab and github 
https://codelabs.developers.google.com/codelabs/cloud-webapp-hosting-gce#5
https://github.com/googlecodelabs/monolith-to-microservices

## git to GCS
https://github.com/Nakilon/git-to-gcs
https://sha.ws/automatic-upload-to-google-cloud-storage-with-github-actions.html

## All networking pricing:  static IP, LB etc
https://cloud.google.com/vpc/network-pricing#ipaddress


## create 2 standalone VM instances
gcloud compute instances create backend \
    --machine-type=f1-micro \
    --image=debian-9-stretch-v20190905 \
    --image-project=debian-cloud \
    --tags=backend \
    --metadata=startup-script-url=https://storage.googleapis.com/fancy-store-qwiklabsproj/startup-script.sh


  gcloud compute instances create frontend \
    --machine-type=f1-micro \
    --image=debian-9-stretch-v20190905 \
    --image-project=debian-cloud \
    --tags=frontend \
    --metadata=startup-script-url=https://storage.googleapis.com/fancy-store-qwiklabsproj/startup-script.sh 

  gcloud compute firewall-rules create fw-fe \
    --allow tcp:8080 \
    --target-tags=frontend

  gcloud compute firewall-rules create fw-be \
    --allow tcp:8081-8082 \
    --target-tags=backend

  // If you stop at this point and choose not to scale the environment, then the instances should be configured to use static IP addresses. You should continue creating managed instance groups and a load balancer even if high scale is not needed so that redundancy and autohealing can be implemented.
  
  
  ## create MIG: 
  gcloud compute instances stop frontend
  gcloud compute instances stop backend

  gcloud compute instance-templates create fancy-fe \
    --source-instance=frontend

  gcloud compute instance-templates create fancy-be \
    --source-instance=backend


  gcloud compute instance-groups managed create fancy-fe-mig \
    --base-instance-name fancy-fe \
    --size 2 \
    --template fancy-fe

  gcloud compute instance-groups managed create fancy-be-mig \
    --base-instance-name fancy-be \
    --size 2 \
    --template fancy-be

  gcloud compute instance-groups set-named-ports fancy-fe-mig --named-ports frontend:8080
  gcloud compute instance-groups set-named-ports fancy-be-mig --named-ports orders:8081,products:8082

  //creating health check
  gcloud compute health-checks create http fancy-fe-hc \
    --port 8080 \
    --check-interval 30s \
    --healthy-threshold 1 \
    --timeout 10s \
    --unhealthy-threshold 3

  gcloud compute health-checks create http fancy-be-hc \
    --port 8081 \
    --request-path=/api/orders \
    --check-interval 30s \
    --healthy-threshold 1 \
    --timeout 10s \
    --unhealthy-threshold 3

  //allow access ports
  gcloud compute firewall-rules create allow-health-check \
    --allow tcp:8080-8081 \
    --source-ranges 130.211.0.0/22,35.191.0.0/16 \
    --network default


  //apply health check to mig
  gcloud compute instance-groups managed update fancy-fe-mig \
    --health-check fancy-fe-hc \
    --initial-delay 300

 gcloud compute instance-groups managed update fancy-be-mig \
    --health-check fancy-be-hc \
    --initial-delay 300

  ## create HTTP Load balancer

  //Create health checks that will be used to determine which instances are capable of serving traffic for each service. 
  gcloud compute http-health-checks create fancy-fe-frontend-hc \
  --request-path / \
  --port 8080

  gcloud compute http-health-checks create fancy-be-orders-hc \
  --request-path /api/orders \
  --port 8081

 gcloud compute http-health-checks create fancy-be-products-hc \
  --request-path /api/products \
  --port 8082

 //Create backend services that are the target for load-balanced traffic. The backend services will use the health checks and named ports that you created.

 gcloud compute backend-services create fancy-fe-frontend \
  --http-health-checks fancy-fe-frontend-hc \
  --port-name frontend \
  --global

 gcloud compute backend-services create fancy-be-orders \
  --http-health-checks fancy-be-orders-hc \
  --port-name orders \
  --global

 gcloud compute backend-services create fancy-be-products \
  --http-health-checks fancy-be-products-hc \
  --port-name products \
  --global

  // add target for Backend services
  //LB --> backend services --> instance groups --> instances
  gcloud compute backend-services add-backend fancy-fe-frontend \
  --instance-group fancy-fe-mig \
  --instance-group-zone us-central1-a \
  --global
  
  gcloud compute backend-services add-backend fancy-be-orders \
  --instance-group fancy-be-mig \
  --instance-group-zone us-central1-a \
  --global

 gcloud compute backend-services add-backend fancy-be-products \
  --instance-group fancy-be-mig \
  --instance-group-zone us-central1-a \
  --global

  ### Create a URL map. The URL map defines which URLs are directed to which backend services.
  gcloud compute url-maps create fancy-map \
  --default-service fancy-fe-frontend
  
  gcloud compute url-maps add-path-matcher fancy-map \
   --default-service fancy-fe-frontend \
   --path-matcher-name orders \
   --path-rules "/api/orders=fancy-be-orders,/api/products=fancy-be-products"

  ### Create the proxy that ties to the created URL map. Create a global forwarding rule that ties a public IP address and port to the proxy.

  gcloud compute target-http-proxies create fancy-proxy \
  --url-map fancy-map
  
  gcloud compute forwarding-rules create fancy-http-rule \
  --global \
  --target-http-proxy fancy-proxy \
  --ports 80
 
  // get the static IP: Find the IP address for the load balancer: 34.120.172.19
  gcloud compute forwarding-rules list --global 
 
  // update end point  configuration:  .env file with new IPs

  //rolling-action: restart 
  gcloud compute instance-groups managed rolling-action restart fancy-fe-mig \
    --max-unavailable 100%

  //rolling-action:  start-update
  gcloud compute instances set-machine-type frontend --machine-type custom-4-3840
  gcloud compute instance-templates create fancy-fe-new \
    --source-instance=frontend \
    --source-instance-zone=us-central1-a
  gcloud compute instance-groups managed rolling-action start-update fancy-fe-mig \
    --version template=fancy-fe-new


  //manual resize  to Zero 
  gcloud compute instance-groups managed resize fancy-fe-mig --size 0


  //auto resizing 
  gcloud compute instance-groups managed set-autoscaling \
  fancy-fe-mig \
  --max-num-replicas 5 \
  --target-load-balancing-utilization 0.60

  //enanble CDN
  gcloud compute backend-services update fancy-fe-frontend \
    --enable-cdn --global

Now, when a user requests content from the load balancer, the request arrives at a Google frontend, which first looks in the Cloud CDN cache for a response to the user's request. If the frontend finds a cached response, then it sends the cached response to the user. That's called a cache hit.

Otherwise, if the frontend can't find a cached response for the request, then it makes a request directly to the backend. If the response to that request is cacheable, then the frontend stores the response in the Cloud CDN cache so that the cache can be used for subsequent requests.




  
