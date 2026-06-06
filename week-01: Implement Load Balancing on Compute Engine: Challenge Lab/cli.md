# Implement Load Balancing on Compute Engine: Challenge Lab


## Setup

```bash
REGION=
ZONE=
```

![Set up your environment](../screenshots/week01/2026-06-06_10-52.png "Set up region and zone")

*Figure 1. Set up your environment.*

## Task 1: Create Web Servers

```bash
# ---------------- Task 1: Create Web Servers -------------------
cat > script1.sh <<'EOF_END'
#!/bin/bash
sudo apt-get update
sudo apt-get install apache2 -y
service apache2 restart
echo "<h3>Web Server: web1</h3>" | tee /var/www/html/index.html
EOF_END

cat > script2.sh <<'EOF_END'
#!/bin/bash
sudo apt-get update
sudo apt-get install apache2 -y
service apache2 restart
echo "<h3>Web Server: web2</h3>" | tee /var/www/html/index.html
EOF_END

cat > script3.sh <<'EOF_END'
#!/bin/bash
sudo apt-get update
sudo apt-get install apache2 -y
service apache2 restart
echo "<h3>Web Server: web3</h3>" | tee /var/www/html/index.html
EOF_END


gcloud compute instances create web1 \
    --zone=$ZONE \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --network=default \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata-from-file startup-script=script1.sh

gcloud compute instances create web2 \
    --zone=$ZONE \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --network=default \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata-from-file startup-script=script2.sh

gcloud compute instances create web3 \
    --zone=$ZONE \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --network=default \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata-from-file startup-script=script3.sh


# Firewall for Network LB
gcloud compute firewall-rules create www-firewall-network-lb \
    --allow tcp:80 \
    --network=default \
    --target-tags=network-lb-tag
```

![Create web servers](../screenshots/week01/2026-06-06_10-53.png "Create web servers")

*Figure 2. Create web servers.*

![Create firewall rules](../screenshots/week01/2026-06-06_11-05.png "Create firewall rules")

*Figure 3. Create firewall rules.*

![Check web servers](../screenshots/week01/2026-06-06_11-06.png "Check web servers")

*Figure 4. Check web servers.*

![Check firewall rules](../screenshots/week01/2026-06-06_11-08.png "Check firewall rules")

*Figure 5. Check firewall rules.*

## Task 2: Network Load Balancer

```bash
# ---------------- Task 2: Network Load Balancer -------------------

gcloud compute addresses create network-lb-ip-1 \
    --region=$REGION

gcloud compute http-health-checks create basic-check

gcloud compute target-pools create www-pool \
    --region=$REGION \
    --http-health-check basic-check

gcloud compute target-pools add-instances www-pool \
    --instances=web1,web2,web3 \
    --zone=$ZONE

gcloud compute forwarding-rules create www-rule \
    --region=$REGION \
    --ports=80 \
    --address=network-lb-ip-1 \
    --target-pool=www-pool
```

![Create Network Load Balancer](../screenshots/week01/2026-06-06_11-09.png "Create Network Load Balancer")

*Figure 6. Create Network Load Balancer.*

![Check address](../screenshots/week01/2026-06-06_11-16.png "Check address")

*Figure 7. Check address.*

![Check health check](../screenshots/week01/2026-06-06_11-17.png "Check health check")

*Figure 8. Check health check.*

![Check load balancer](../screenshots/week01/2026-06-06_11-18.png "Check load balancer")

*Figure 9. Check load balancer.*

![Check forwarding rule](../screenshots/week01/2026-06-06_11-19.png "Check forwarding rule")

*Figure 10. Check forwarding rule.*

## Task 3: HTTP Load Balancer

```bash
# ---------------- Task 3: HTTP Load Balancer -------------------


cat > script4.sh <<'EOF_END'
#!/bin/bash
sudo apt-get update
sudo apt-get install apache2 -y
vm=\$(hostname)
echo \"Page served from: \$vm\" > /var/www/html/index.html
systemctl restart apache2
EOF_END

# Instance Template
gcloud compute instance-templates create lb-backend-template \
  --machine-type=e2-medium \
  --tags=allow-health-check \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file startup-script=script4.sh

# Managed Instance Group
gcloud compute instance-groups managed create lb-backend-group \
  --template=lb-backend-template \
  --size=2 \
  --zone=$ZONE

# Firewall for HC
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80

# Global IP
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global

# Health check
gcloud compute health-checks create http http-basic-check --port=80

# Backend service
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global

gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=$ZONE \
  --global

# URL Map & Proxy
gcloud compute url-maps create web-map-http \
  --default-service web-backend-service

gcloud compute target-http-proxies create http-lb-proxy \
  --url-map=web-map-http

# Forwarding rule
gcloud compute forwarding-rules create http-content-rule \
  --address=lb-ipv4-1 \
  --global \
  --target-http-proxy=http-lb-proxy \
  --ports=80
```

![Create instance template](../screenshots/week01/2026-06-06_11-15.png "Create instance template")

*Figure 11. Create instance template.*

![Check instance template](../screenshots/week01/2026-06-06_11-21.png "Check instance template")

*Figure 12. Check instance template.*

![Create instance groups](../screenshots/week01/2026-06-06_11-22.png "Create instance groups")

*Figure 13. Create instance groups.*

![Check instance groups](../screenshots/week01/2026-06-06_11-23.png "Check instance groups")

*Figure 14. Check instance groups.*

![Create firewall rules](../screenshots/week01/2026-06-06_11-24.png "Create firewall rules")

*Figure 15. Create firewall rules.*

![Check firewall rules](../screenshots/week01/2026-06-06_11-25.png "Check firewall rules")

*Figure 16. Check firewall rules.*

![Create address](../screenshots/week01/2026-06-06_11-26.png "Create address")

*Figure 17. Create address.*

![Check address](../screenshots/week01/2026-06-06_11-27.png "Check address")

*Figure 18. Check address.*

![Create health check and web backend service](../screenshots/week01/2026-06-06_11-28_1.png "Create health check and web backend service")

*Figure 19. Create health check and web backend service.*

![Check health check](../screenshots/week01/2026-06-06_11-28.png "Check health check")

*Figure 20. Check health check.*

![Check web backend service](../screenshots/week01/2026-06-06_11-28_2.png "Check web backend service")

*Figure 21. Check web backend service.*

![Detect web backend service error](../screenshots/week01/2026-06-06_11-29.png "Detect web backend service error")

*Figure 22. Detect web backend service error.*

![Create url-map, proxy, and forwarding rule](../screenshots/week01/2026-06-06_11-30.png "Create url-map, proxy, and forwarding rule")

*Figure 23. Create url-map, proxy, and forwarding rule.*

![Check url-map, proxy, and forwarding rule](../screenshots/week01/2026-06-06_11-31.png "Check url-map, proxy, and forwarding rule")

*Figure 24. Check url-map, proxy, and forwarding rule.*
