# Access Token Broker

This repository contains the sample code for the 
[Authenticating to services by using client certificates](https://cloud.google.com/solutions/using-mutual-tls-to-obtain-short-lived-credentials) tutorial.

# Configuration Steps
[Tutorial Page](https://cloud.google.com/solutions/using-mutual-tls-to-obtain-short-lived-credentials)

## Set Environment
```
gcloud config set project <project id>
gcloud config set compute/region us-east4 && \
	gcloud config set compute/zone us-east4-a && \
	STORAGE_READER_SA=storage-reader@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com && \
	TOKEN_BROKER_SA=token-broker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```
## Set up the server
```
gcloud compute instances create token-broker \
  --service-account=$TOKEN_BROKER_SA \
  --tags=https,ssh \
	--machine-type g1-small \
	--image-project debian-cloud \
	--image-family debian-9 \
  --scopes https://www.googleapis.com/auth/cloud-platform \
	--network token-broker-vpc \
	--subnet subnet1 \
  --metadata startup-script='#! /bin/bash
        sudo apt update -y
        sudo apt install git python3-pip -y

        sudo git clone https://github.com/GoogleCloudPlatform/python-token-broker.git /opt/python-token-broker
        sudo pip3 install -r /opt/python-token-broker/requirements.txt'

TOKEN_BROKER_FQDN=$(gcloud compute instances describe token-broker --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
gcloud compute scp clientcas.crt token-broker.crt token-broker:~
gcloud compute ssh token-broker --command 'sudo mv clientcas.crt token-broker.crt /opt/python-token-broker'
gcloud compute ssh token-broker --command "(cd /opt/python-token-broker && sudo python3 token_broker.py 0.0.0.0 443 token-broker.crt  $STORAGE_READER_SA clientcas.crt)"
```
## Test the build
```
$ curl \
    --cert client.crt --key client.key \
    -k \
    https://$TOKEN_BROKER_FQDN/token

$ TOKEN=$(curl --cacert ./ca.crt --key client.key --cert ./client.crt https://$TOKEN_BROKER_FQDN/token)

$ curl \
    -H 'Accept: application/json' \
    -H "Authorization: Bearer ${TOKEN}" \
    -k \
    https://storage.googleapis.com/storage/v1/b/$GOOGLE_CLOUD_PROJECT-storage/o
```
## License

All files in this repository are under the
[Apache License, Version 2.0](LICENSE.txt) unless noted otherwise.
