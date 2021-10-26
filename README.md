# go-wafsecurity-test

GigaOm WAF Security Test is a Web Application Firewall performance test utilizing github.com/tsenart/vegeta.

**Components**

The field test consists of the following components:

- [Backend API](#backendapi)
- [Attack Client](#attackclient)
- [NGINX](#nginx)
- [AWS WAF](#awswaf)
- [Azure WAF](#azurewaf)
- [Test Sequence](#testsequence)

---

## Backend API

To build the backend API, we used an AWS EC2 c5n.xlarge Instance running Ubuntu 20.04.2 LTS (Focal Fossa).

**Step 1.** We increased the maximum number of open files:
```bash
sudo vim /etc/security/limits.conf
```
We added this to the end of the file:
```bash
* soft nofile 65536
* hard nofile 65536
```
Log out and back in
**Step 2.** We installed open-source NGINX:
```bash
sudo apt update
sudo apt install nginx
```
**Step 3.** Backup and copy the new NGINX config (from this repo):
```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bkp
sudo cp api.nginx.conf /etc/nginx/nginx.conf
sudo systemctl restart nginx
```
**Step 3b.** [optional] Generate your own random string:
```bash
python
from os import urandom
from base64 import b64encode
print(b64encode(urandom(1024)).decode('utf-8'))
exit()
```
**Step 4.** Test the backend API is working:
```bash
curl 127.0.0.1:1980
```

## Attack Client

To build the backend API, we used an AWS EC2 c5n.2xlarge Instance running Ubuntu 20.04.2 LTS (Focal Fossa).

**Step 1.** We increased the maximum number of open files:
```bash
sudo vim /etc/security/limits.conf
```
We added this to the end of the file:
```bash
* soft nofile 65536
* hard nofile 65536
```
Log out and back in
**Step 2.** We installed Go and Vegeta:
```bash
sudo apt update
sudo apt install go --classic
go --version
go get -u github.com/tsenart/vegeta
go/bin/vegeta --help
```
**Step 3.** Attack the backend API directly:
```bash
curl <backendapi-internal-ip>:1980
echo "GET http://172.31.11.74:1980" | go/bin/vegeta attack -duration=10s -rate=1000/1s | tee results-direct-1000rate.bin | go/bin/vegeta report
```

## NGINX

To build the 3 NGINX nodes, we used AWS EC2 c5n.xlarge Instances running CentOS 7.6.

**Step 1.** We increased the maximum number of open files:
```bash
sudo vim /etc/security/limits.conf
```
We added this to the end of the file:
```bash
* soft nofile 65536
* hard nofile 65536
```
Log out and back in
**Step 2.** We secure copied to the instances our NGINX certificate and key and put them in place:
```bash
sudo mkdir -p /etc/ssl/nginx
sudo cp nginx-repo.* /etc/ssl/nginx/
```
**Step 3.** We install pre-requisites and changed SELinux settings:
```bash
sudo setsebool -P httpd_can_network_connect true
sudo yum update
sudo yum install vim ca-certificates epel-release
```
**Step 4.** We install repos, NGINX Plus, security rules, and new configs:
No Security
```bash
sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/nginx-plus-7.4.repo
sudo yum install nginx-plus
nginx -v
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bkp
sudo cp nosec.nginx.conf /etc/nginx/nginx.conf
sudo systemctl restart nginx
```
Mod Security
```bash
sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/nginx-plus-7.4.repo
sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/modsecurity-7.repo
sudo yum install nginx-plus nginx-plus-module-modsecurity
nginx -v
wget https://github.com/SpiderLabs/owasp-modsecurity-crs/archive/v3.0.2.tar.gz
tar -xzvf v3.0.2.tar.gz
sudo mkdir /etc/nginx/modsec/owasp/
sudo cp owasp-modsecurity-crs-3.0.2/rules/*.conf /etc/nginx/modsec/owasp
sudo cp owasp-modsecurity-crs-3.0.2/crs-setup.conf.example /etc/nginx/modsec/crs-setup.conf
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bkp
sudo cp modsec.nginx.conf /etc/nginx/nginx.conf
sudo cp modsec.main.conf /etc/nginx/modsec/main.conf
sudo vim /etc/nginx/modsec/main.conf
sudo systemctl restart nginx
```
AppProtect WAF
```bash
sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/nginx-plus-7.4.repo
sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/app-protect-7.repo
sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/app-protect-security-updates-7.repo
sudo yum install nginx-plus app-protect app-protect-attack-signatures app-protect-threat-campaigns
nginx -v
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bkp
sudo cp appp.nginx.conf /etc/nginx/nginx.conf
sudo systemctl restart nginx
```
**Step 5.** Test that security is working properly:
```bash
curl "http://localhost/"
curl "http://localhost?a=<script>"
```

## AWS WAF

To setup AWS WAF, we followed their public documentation at https://docs.aws.amazon.com/waf/index.html

We used the AWS-AWSManagedRulesCommonRuleSet rules.

## Azure WAF

To setup Azure WAF, we followed their public documentation at https://docs.microsoft.com/en-us/azure/web-application-firewall/

We used the Azure Core Rule Set OWASP 3.2 https://docs.microsoft.com/en-us/azure/web-application-firewall/ag/application-gateway-crs-rulegroups-rules?tabs=owasp32

**NOTE:** We also setup Azure Virtual Machines following the same Backend API and Attack instructions above, so there would closer network proximity between Azure WAF and the other components.

## Test Sequence

To execute the tests, we used the following bash scripts. You are welcome to design your own test sequences.

To test throughput:
```bash
#!/bin/bash
declare -A endpoints=(
["modsec"]="http://modsec-endpoint/" ["appp"]="http://appprotect-endpoint/" ["awswaf"]="https://awswaf-endpoint")
rate=1000
for platform in "${!endpoints[@]}"
  do echo
  echo "Attacking: $platform @ ${endpoints[$platform]}"; echo "GET ${endpoints[$platform]}" | go/bin/vegeta attack -duration=10s -rate=${rate}/1s | tee results-${platform}-good-${rate}rate.bin | go/bin/vegeta report -type=hdrplot | grep '0.500000\|0.900000\|0.950000\|0.990\|0.9990\|0.99990\|1.000000' &
  sleep 3
done
```

To test "good" and "bad" traffic simultaneously:
```bash
#!/bin/bash
declare -A endpoints=(
["modsec"]="http://modsec-endpoint/" ["appp"]="http://appprotect-endpoint/" ["awswaf"]="https://awswaf-endpoint")
rate=450
badrate=50
for platform in "${!endpoints[@]}"
  do echo
  echo "Attacking Good: $platform @ ${endpoints[$platform]}"; echo "GET ${endpoints[$platform]}" | go/bin/vegeta attack -duration=10s -rate=${rate}/1s | tee results-${platform}-good-${rate}rate.bin | go/bin/vegeta report -type=hdrplot | grep '0.500000\|0.900000\|0.950000\|0.990\|0.9990\|0.99990\|1.000000' &

  echo "Attacking Bad : $platform @ ${endpoints[$platform]}"; echo "GET ${endpoints[$platform]}?a=<script>" | go/bin/vegeta attack -duration=10s -rate=${badrate}/1s | tee results-${platform}-bad-${badrate}rate.bin | go/bin/vegeta report -type=hdrplot | grep '0.500000\|0.900000\|0.950000\|0.990\|0.9990\|0.99990\|1.000000'
  sleep 3
done
```


