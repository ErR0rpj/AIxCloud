# Creating VM in Google cloud

2 ways:

1. G cloud console UI
2. G cloud cmd line

## Creating VM with cmd line

### Prequisites

Complete the google-cmd-line-tools.md for region set

### Creating VM with CMD line

gcloud compute instances create gcelab2 --machine-type e2-medium --zone=$ZONE
gcloud compute instances create --help
gcloud compute ssh gcelab2 --zone=$ZONE

### Installing nginx in VM

Login to host wither by clicking on ssh button in the console or by loggin in with: gcloud compute ssh gcelab2 --zone=$ZONE in CMD line
sudo apt-get update
sudo apt-get install -y nginx
ps auwx | grep nginx
http://EXTERNAL_IP/

After all the above configs the VM will not be reachable with the external IP still as the https and http firewalls are blocked and we can enable those from console by selecting the tags for http and https traffic or by command line can be found in cmd line file.
