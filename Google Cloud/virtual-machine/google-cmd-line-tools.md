# Google Command Line tools

## Configuration Commands

gcloud auth list
gcloud config list project
gcloud config set compute/region us-east4
gcloud config set compute/zone us-east4-a
export REGION=us-east4
export ZONE=us-east4-a

## VM: Instances, Firewall, Logs, Load balancing

### Instance commands

#### Creation of instance

gcloud compute instances create gcelab2 --machine-type e2-medium --zone=$ZONE
gcloud compute instances create --help
gcloud compute ssh gcelab2 --zone=$ZONE

#### Listing Instaces

gcloud compute instances list
gcloud compute instances list --filter="name=('gcelab2')"

#### Installing nginx in VM

Login to host wither by clicking on ssh button in the console or by loggin in with: gcloud compute ssh gcelab2 --zone=$ZONE in CMD line
sudo apt-get update
sudo apt-get install -y nginx
ps auwx | grep nginx
http://EXTERNAL_IP/

After all the above configs the VM will not be reachable with the external IP still as the https and http firewalls are blocked and we can enable those from console by selecting the tags for http and https traffic or by command line can be found in cmd line file.

### Firewall commands

gcloud compute firewall-rules list
gcloud compute firewall-rules list --filter="NETWORK:'default' AND ALLOW:'icmp'"
// Adding network traffic tags
gcloud compute instances add-tags gcelab2 --tags http-server,https-server
// Updating firewall rules to allow
gcloud compute firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
gcloud compute firewall-rules list --filter=ALLOW:'80'
// To check if connection is established with external networks
curl http://$(gcloud compute instances list --filter=name:gcelab2 --format='value(EXTERNAL_IP)')

### Logs

gcloud logging logs list
gcloud logging logs list --filter="compute"
gcloud logging read "resource.type=gce_instance" --limit 5
gcloud logging read "resource.type=gce_instance AND labels.instance_name='gcelab2'" --limit 5

### Creating VM with image, with Apache installation, firewalls allowed and unique home page

// Repalce () with <> for html
gcloud compute instances create www1 \
    --zone=europe-west1-c \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
(h3)Web Server: www1</h3>" | tee /var/www/html/index.html'

### Load Balancing configuration

#### Configuration and forwarding for load balancer service

// Creates the static external IP for load balancer
gcloud compute addresses create network-lb-ip-1 \
  --region europe-west1
// Add leagcy http health check resource
gcloud compute http-health-checks create basic-check
// Creates the target pool in same region of instance and Health check
gcloud compute target-pools create www-pool \
  --region europe-west1 --http-health-check basic-check
// Adding the instaces to the pool. Here, names of VMs were: www1, www2, www3
gcloud compute target-pools add-instances www-pool \
    --instances www1,www2,www3
// Adding the forwarding rules to the above load balancer IP and pool of instaces
gcloud compute forwarding-rules create www-rule \
    --region  europe-west1 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool

#### Sending traffic

gcloud compute forwarding-rules describe www-rule --region europe-west1
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region europe-west1 --format="json" | jq -r .IPAddress)
echo $IPADDRESS
while true; do curl -m1 $IPADDRESS; done

#### HTTP Load balancer

// Creating load balancer template
gcloud compute instance-templates create lb-backend-template \
   --region=europe-west1 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
// Creating MIG group
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=europe-west1-c
// Creating firewall rules
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
// Making IP public
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
// Note the IP address
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global 34.36.127.78
// Health check for Load balancer
gcloud compute health-checks create http http-basic-check \
  --port 80
// Creating backend service
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
// Adding instances to the backend service
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=europe-west1-c \
  --global
// Creaitng url map to map the requests to the service
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
// Creating proxy to route the requests to URL map
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
// Creating global forwarding rule to route the requests to proxy
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
