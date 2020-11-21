# Docs
https://codelabs.developers.google.com/codelabs/cloud-deploy-website-on-gke#10

##

### create
gcloud container clusters create fancy-cluster --num-nodes 2

### get credential
gcloud container clusters get-credentials fancy-cluster

### resize Kube cluster
gcloud container clusters resize fancy-cluster --num-nodes=0
/// resize MIG
gcloud compute instance-groups managed resize fancy-fe-mig --size 0

### enable cloudbuild 
gcloud services enable cloudbuild.googleapis.com

### build src
cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 .




### create deployment from images 
kubectl create deployment monolith --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0

### review deployments 
kubectl describe pod monolith
kubectl describe pod/monolith-7d8bc7bf68-2bxts
kubectl describe deployment monolith
kubectl describe deployment.apps/monolith

//Show pods
kubectl get pods

//Show deployments
kubectl get deployments

// Show replica sets
kubectl get rs

#You can also combine them
kubectl get pods,deployments

### similate pod crash
kubectl delete pod/<POD_NAME>

### expose deployment as service
kubectl expose deployment monolith --type=LoadBalancer --port 80 --target-port 8080
kubectl get service

### scale NODEs and PODs

kubectl scale deployment monolith --replicas=3
gcloud container clusters resize fancy-cluster --num-nodes=3

### update code
change index.js,   add a date

cd ~/monolith-to-microservices/react-app
npm run build:monolith
cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0 .

//update image 
kubectl set image deployment/monolith monolith=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0




