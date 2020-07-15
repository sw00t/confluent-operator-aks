# Confluent Platform 5.5 Operator on AKS / Azure Kubernetes Service demo [WIP]

Last updated: 9 July 2020

## Prerequisites
Install Azure CLI. MacOS:
```
brew install az
```

Login to Azure
```
az login
```

# Deploy an AKS cluster
Check K8s version availability for the selected zone. Comparison table per region at the link below:

https://azure.microsoft.com/en-us/global-infrastructure/services/?regions=us-east,us-west,us-west-2&products=all
```
az aks get-versions --location westus --output table
```

## Export resource group and AKS cluster
```
export RG=swoo-rg
```
```
export CLUSTER=swooAKSdemo
```

## Create AKS cluster (~5min)
/Users/swoo/.ssh/id_rsa and /Users/swoo/.ssh/id_rsa.pub generated
```
az aks create \
  --resource-group $RG \
  --name $CLUSTER \
  --kubernetes-version 1.18.2 \
  --node-count 3 \
  --load-balancer-sku basic \
  --disable-rbac \
  --ssh-key-value ~/.ssh/id_rsa.pub
```

If no keys are created/available, generate them by including this in the above command:
```
--generate-ssh-keys
```


## Connect kubectl to AKS
```
az aks get-credentials --resource-group $RG --name $CLUSTER
```

## Verify
```
kubectl cluster-info
kubectl get no -owide
```

# Get, Prep, and Deploy Confluent Platform
Download & unpack Confluent Operator
```
wget https://platform-ops-bin.s3-us-west-1.amazonaws.com/operator/confluent-operator-5.5.0.tar.gz
tar -xvf confluent-operator-5.5.0.tar.gz
cd confluent-operator/
```

Export Operator directory, and Operator values configuration file
```
export OPAKS=/Users/swoo/0/Confluent/Operator
```
TODO: clean this section up
```
export MYVALUESFILE=/Users/swoo/0/sw00t/confluent-operator/azure/swvalues-aks.yaml
```

Copy the Azure values configuration file to new
In new values file, update
-region and zones
-enable and configure load balancer for external access

## Create namespace 'confluent', set to current context
```
kubectl create ns confluent && kubectl config set-context --current --namespace confluent
```

## Helm install for Operator, ZooKeeper, Kafka, and Control Center
```
helm install operator $OPAKS/helm/confluent-operator \
  --values $MYVALUESFILE \
  --namespace confluent \
  --set operator.enabled=true
```
sleep 10s
```
helm install zookeeper $OPAKS/helm/confluent-operator \
 --values $MYVALUESFILE \
 --namespace confluent \
 --set zookeeper.enabled=true
```
sleep 30s
```
helm install kafka $OPAKS/helm/confluent-operator \
   --values $MYVALUESFILE \
   --namespace confluent \
   --set kafka.enabled=true
```
sleep 30s
```
helm install controlcenter $OPAKS/helm/confluent-operator \
  --values $MYVALUESFILE \
  --namespace confluent \
  --set controlcenter.enabled=true
```
sleep 15s

## Watch pods spin up until READY status is 1/1 for all pods (~17min)
```
watch kubectl get po
```

## In a new term window, expose Control Center
```
kubectl -n confluent port-forward controlcenter-0 12345:9021
open http://localhost:12345 # admin/Developer1
```

# Configure DNS

Set the DNS zone
```
export DNSzone=swoodemo.dev
```

Run, then scroll to 7. Client Access:, External:
```
kubectl -n confluent get kafka kafka  -oyaml
```

Get the kafka bootstrap LB external IP
```
kubectl get svc | grep kafka-bootstrap-lb
```
```
export bootstrap=<IP>
```

Get the domain, and LB endpoint IPs
```
kubectl -n confluent get kafka kafka -oyaml | grep "    domain: "
kubectl -n confluent describe svc kafka-0-lb | grep "LoadBalancer Ingress"
kubectl -n confluent describe svc kafka-1-lb | grep "LoadBalancer Ingress"
kubectl -n confluent describe svc kafka-2-lb | grep "LoadBalancer Ingress"
```

Create Azure DNS records for the domain
```
az network dns zone create -g $RG -n $DNSzone
```

Create Azure DNS records for kafka
```
az network dns record-set a add-record -g $RG -z $DNSzone -n kafka -a <IP from above>
```

Run the following to create Azure DNS records for broker endpoints
```
az network dns record-set a add-record -g $RG -z $DNSzone -n b0 -a <IP from above>
az network dns record-set a add-record -g $RG -z $DNSzone -n b1 -a <IP from above>
az network dns record-set a add-record -g $RG -z $DNSzone -n b2 -a <IP from above>
```

# <insert test stuff here>


# Clean up
```
helm delete controlcenter && helm delete kafka && helm delete zookeeper && helm delete operator
```
```
az dns delete --resource-group $RG --name $DNSzone
```
```
az aks delete --resource-group $RG --name $CLUSTER -y
```
```
az group delete --name $RG
```


