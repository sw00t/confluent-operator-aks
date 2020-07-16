# Confluent Platform 5.5 Operator on AKS / Azure Kubernetes Service demo [WIP]

Last updated: 16 July 2020

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

## Open a new term window and run a Watch to observe pods spin up in following steps
READY 1/1 status for all pods should total ~12min.
```
watch kubectl get po
```

## Helm install for Operator, ZooKeeper, Kafka, and Control Center
```
helm install operator $OPAKS/helm/confluent-operator \
  --values $MYVALUESFILE \
  --namespace confluent \
  --set operator.enabled=true
```
sleep 1m
```
helm install zookeeper $OPAKS/helm/confluent-operator \
 --values $MYVALUESFILE \
 --namespace confluent \
 --set zookeeper.enabled=true
```
sleep 2m
```
helm install kafka $OPAKS/helm/confluent-operator \
   --values $MYVALUESFILE \
   --namespace confluent \
   --set kafka.enabled=true
```
sleep 5m
```
helm install controlcenter $OPAKS/helm/confluent-operator \
  --values $MYVALUESFILE \
  --namespace confluent \
  --set controlcenter.enabled=true
```
sleep 2m

# Optional Helm installs
Additional components can be installed, however will require scaling of AKS cluster resources. For example:
```
az aks scale --resource-group $RG --name $CLUSTER --node-count 5 --nodepool-name nodepool1
```

## Schema Registry
```
helm upgrade --install schemaregistry $OPAKS/helm/confluent-operator --values $MYVALUESFILE --namespace confluent --set schemaregistry.enabled=true
```
## Kafka Connect
```
helm upgrade --install connectors $OPAKS/helm/confluent-operator --values $MYVALUESFILE --namespace confluent --set connect.enabled=true
```
## Confluent Replicator
```
helm upgrade --install replicator $OPAKS/helm/confluent-operator --values $MYVALUESFILE --namespace confluent --set replicator.enabled=true
```
## ksqlDB
```
helm upgrade --install ksql $OPAKS/helm/confluent-operator --values $MYVALUESFILE --namespace confluent --set ksql.enabled=true
```


## In a new term window, expose Control Center
```
kubectl -n confluent port-forward controlcenter-0 12345:9021
open http://localhost:12345 # admin/Developer1
```

# Configure DNS

## Set the DNS zone
```
export DNSzone=swoodemo.dev
```
## Create Azure DNS zone for the domain
```
az network dns zone create -g $RG -n $DNSzone
```

## Get the kafka bootstrap LB external IP
```
kubectl get svc | grep kafka-bootstrap-lb
export bootstrap=<above IP>
```
## Create Azure DNS Zone records for kafka
```
az network dns record-set a add-record -g $RG -z $DNSzone -n kafka -a $bootstrap
```

## Get the LB endpoint IPs, then create Azure DNS records
```
kubectl -n confluent describe svc kafka-0-lb | grep "LoadBalancer Ingress"
az network dns record-set a add-record -g $RG -z $DNSzone -n b0 -a <IP from above>
```
```
kubectl -n confluent describe svc kafka-1-lb | grep "LoadBalancer Ingress"
az network dns record-set a add-record -g $RG -z $DNSzone -n b1 -a <IP from above>
```
```
kubectl -n confluent describe svc kafka-2-lb | grep "LoadBalancer Ingress"
az network dns record-set a add-record -g $RG -z $DNSzone -n b2 -a <IP from above>
```

# Verify entries
```
az network dns record-set list --resource-group $RG --zone-name $DNSzone
```


# Test
## Validate internal access
Ref: https://docs.confluent.io/current/installation/operator/co-deployment.html#internal-access-validation

Update client properties file
```
echo $bootstrap
```
Update bootstrap.servers field in client-aks.properties
```
vi ~/0/sw00t/confluent-operator/azure/client-aks.properties
```

## Create topic [WIP]
kafka-topics --bootstrap-server $bootstrap:9092 \
  --command-config client-aks.properties \
  --create --topic test \
  --partitions 1 \
  --replication-factor 1

## Produce to topic from within k8s pod [WIP]
## Get info (including internal bootstrap server)
  kubectl get kafka -n confluent -oyaml
## ssh into a broker pod:
  kubectl -n confluent exec -it kafka-0 bash
## create properties file in the pod:
  cat << EOF > kafka.properties
  bootstrap.servers=kafka:9071
  sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="test" password="test123";
  sasl.mechanism=PLAIN
  security.protocol=SASL_PLAINTEXT
  EOF
## query bootstrap server
  kafka-broker-api-versions --command-config kafka.properties --bootstrap-server kafka:9071
  #exit the server
  exit

## Validate external access
ref: https://docs.confluent.io/current/installation/operator/co-deployment.html#external-access-validation


# <Optional> Upgrade AKS cluster version
Check what versions are available to upgrade to:
```
az aks get-upgrades --resource-group $RG --name $CLUSTER --output table
```
Upgrade cluster version
```
az aks upgrade --resource-group $RG --name $CLUSTER --kubernetes-version <KUBERNETES_VERSION>
```
Example:
```
az aks upgrade --resource-group $RG --name $CLUSTER --kubernetes-version 1.18.4
```


# Clean up
```
helm delete controlcenter && helm delete kafka && helm delete zookeeper && helm delete operator
```
```
az network dns zone delete --resource-group $RG --name $DNSzone -y
```
```
az aks delete --resource-group $RG --name $CLUSTER -y
```
```
az group delete --name $RG -y
```
