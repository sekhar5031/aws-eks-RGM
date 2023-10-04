
STEPS: 
----------------
Install Kubectl:
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH


Install AWScli:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

command to set aws context:
aws eks update-kubeconfig --name EKS_CLUSTER_NAME --region us-west-2

clone the AWS repo
git clone

Create Nepaltech Namespace
kubectl create ns nepaltech

set default namespace to nepaltech
kubectl config set-context --current --namespace nepaltech

------------------
MONGO
-------------------
MONGO Database Setup
To create Mongo statefulset with Persistent volumes, run the command in manifests folder:
kubectl apply -f mongo-statefulset.yaml

Mongo Service
kubectl apply -f mongo-service.yaml



Run the following to enter inside the mongo-0 and open mongo shell:
kubectl exec -it mongo-0 -- mongo

On the mongo-0 pod, initialise the Mongo database Replica set. In the terminal run the following command:
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);

check if the replica set is implemented
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"

Load the Data in the database by running this command:

enter inside the db
kubectl exec -it mongo-0 -- mongo

use langdb database:
use langdb;

query to insert the data into the DB.
db.languages.insert({"name" : "csharp", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 5, "compiled" : false, "homepage" : "https://dotnet.microsoft.com/learn/csharp", "download" : "https://dotnet.microsoft.com/download/", "votes" : 0}});
db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 12, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 1, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 20, "script" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});


Create Mongo secret:
kubectl apply -f mongo-secret.yaml

------------
API SETUP
------------------
Create API go deployment:
kubectl apply -f api-deployment.yaml

expose the deployment through service
kubectl expose deploy api \
 --name=api \
 --type=LoadBalancer \
 --port=80 \
 --target-port=8080

 set the service endpoint over the environment variable
 {
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $API_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl $API_ELB_PUBLIC_FQDN/ok
echo
}

Test and confirm that the API route URL /languages, and /languages/{name} endpoints can be called successfully. In the terminal run any of the following commands:
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .


----------------------
FRONTEND SETUP
----------------------

Create the Frontend Deployment resource. 
kubectl apply -f frontend-deployment.yaml

Create a new Service resource of LoadBalancer type

kubectl expose deploy frontend \
 --name=frontend \
 --type=LoadBalancer \
 --port=80 \
 --target-port=8080

 Confirm that the Frontend ELB is ready to recieve HTTP traffic. 

{
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $FRONTEND_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl -I $FRONTEND_ELB_PUBLIC_FQDN
}

Generate the Frontend URL for browsing. In the terminal run the following command:

echo http://$FRONTEND_ELB_PUBLIC_FQDN


Query the MongoDB database directly to observe the updated vote data. In the terminal execute the following command:

kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"