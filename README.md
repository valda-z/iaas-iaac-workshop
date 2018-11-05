# Workshop IaaS / IaaC

In this workshop we will create small infrastructure solution which contains few building blocks:

* resource group and VNET
* jumpbox server (CentOS based)
* application VM with internal IP address and running docker container (started as service)
* Application Gateway - SSL endpoint, routing to application server
* Virtual Machine scale set with load balancer and exposed public IP

## Create support resources

### Create resource group

```bash
# create resource group
export RESOURCE_GROUP="QTEST"
export LOCATION="northeurope"
az group create -l ${LOCATION} -n ${RESOURCE_GROUP}
```

### Create network

```bash
export VNET="vnet"
export SUBNETJUMPBOX="jumpbox"
export SUBNETAPP="app"
export SUBNETVMSS="vmss"
export SUBNETGW="gateway"

# Create VNET
az network vnet create -g ${RESOURCE_GROUP} -n ${VNET} --address-prefix 10.0.0.0/16 \
                            --subnet-name ${SUBNETJUMPBOX} --subnet-prefix 10.0.100.0/24

# Create subnets
az network vnet subnet create -g ${RESOURCE_GROUP} --vnet-name ${VNET} -n ${SUBNETGW} \
                            --address-prefix 10.0.0.0/24
az network vnet subnet create -g ${RESOURCE_GROUP} --vnet-name ${VNET} -n ${SUBNETAPP} \
                            --address-prefix 10.0.10.0/24
az network vnet subnet create -g ${RESOURCE_GROUP} --vnet-name ${VNET} -n ${SUBNETVMSS} \
                            --address-prefix 10.0.11.0/24
```

### Create jumpbox

```bash
# my public ssh-key
export MYSSHKEY="xxxxxxxxxxxx"

# create cloud-init.txt
echo "#cloud-config
package_upgrade: true
packages:
  - curl
  - [git, gcc, libffi-devel, python-devel, openssl-devel, epel-release]
runcmd:
  - yum install -y git gcc libffi-devel python-devel openssl-devel epel-release
  - yum install -y python-pip python-wheel
  - pip install ansible[azure]
" > cloud-init.txt

# create VM
az vm create --name jumpbox \
  --resource-group ${RESOURCE_GROUP} \
  --admin-username myadmin \
  --authentication-type ssh \
  --location ${LOCATION} \
  --no-wait \
  --nsg-rule SSH \
  --image "OpenLogic:CentOS:7-CI:latest" \
  --vnet-name ${VNET} \
  --subnet ${SUBNETJUMPBOX} \
  --size Standard_D1_v2 \
  --ssh-key-value "${MYSSHKEY}" \
  --custom-data cloud-init.txt
```

## Create App server with Application Gateway

Rules for application gateway will be created by in GUI (portal).

```bash
# create application gateway
az network application-gateway create -g ${RESOURCE_GROUP} -n appgw --capacity 1 --sku Standard_Medium \
                            --vnet-name ${VNET} --subnet ${SUBNETGW} --no-wait \
                            --public-ip-address appgw-PublicIp

```

Now lets create VM with private IP (static) and with started docker service.

```bash
# Create NIC for VM
az network nic create \
--resource-group ${RESOURCE_GROUP} \
--name app-nic \
--location ${LOCATION} \
--subnet ${SUBNETAPP} \
--private-ip-address 10.0.10.100 \
--vnet-name ${VNET}

# create cloud-init-app.txt
echo "#cloud-config
package_upgrade: true
packages:
  - curl
  - docker
write_files:
  - content: |
        [Unit]
        Description=MyApp
        After=docker.service
        Requires=docker.service
        [Service]
        TimeoutStartSec=0
        Restart=always
        ExecStartPre=-/usr/bin/docker stop %n
        ExecStartPre=-/usr/bin/docker rm %n
        ExecStartPre=/usr/bin/docker pull dockercloud/hello-world
        ExecStart=/usr/bin/docker run --rm -p 8080:80 --name %n dockercloud/hello-world
        [Install]
        WantedBy=multi-user.target
    path: /etc/systemd/system/docker.myapp.service
runcmd:
  - systemctl enable docker.myapp.service
  - systemctl restart docker.myapp
" > cloud-init-app.txt


# Create App VM
az vm create --name app \
  --resource-group ${RESOURCE_GROUP} \
  --admin-username myadmin \
  --authentication-type ssh \
  --location ${LOCATION} \
  --no-wait \
  --image "OpenLogic:CentOS:7-CI:latest" \
  --nics app-nic \
  --size Standard_D1_v2 \
  --ssh-key-value "${MYSSHKEY}" \
  --custom-data cloud-init-app.txt

```

### Connect to app server

```bash
eval $(ssh-agent)
ssh-add
ssh -A <JumpboxIP>

# test http
curl http://10.0.10/100:8080

# ssh into app server
ssh myadmin@10.0.10.100
```

### Configure Application Gateway

These scripts configure Application gateway to route traffic from https endpoint to backend http service.

```bash
# configure backend servers
az network application-gateway address-pool create \
  --no-wait
  --resource-group ${RESOURCE_GROUP} \
  --gateway-name appgw \
  -n MyAddressPool \
  --servers 10.0.10.100

# Create SSL cert (in folder where mycert.pfx is located)
az network application-gateway ssl-cert create \
  --no-wait \
  --resource-group ${RESOURCE_GROUP} \
  --gateway-name appgw \
  -n MySslCert \
  --cert-file mycert.pfx \
  --cert-password "pwd123..."

# Create FronEnd port (have to wait for load SSL cert)
az network application-gateway frontend-port create \
  --no-wait \
  --resource-group ${RESOURCE_GROUP} \
  --gateway-name appgw \
  -n MyFrontendPort --port 443

# Create HTTP listener for 443
az network application-gateway http-listener create \
  --no-wait \
  --resource-group ${RESOURCE_GROUP} \
  --gateway-name appgw \
  --frontend-port MyFrontendPort \
  -n MyHttpListener \
  --frontend-ip appGatewayFrontendIP \
  --ssl-cert MySslCert

# Create HTTP setting
az network application-gateway http-settings create\
  --no-wait \
  --resource-group ${RESOURCE_GROUP} \
  --gateway-name appgw \
  -n MyHttpSettings --port 8080 \
  --protocol Http --cookie-based-affinity Disabled --timeout 30

# Create routing rule
az network application-gateway rule create \
  --no-wait \
  --resource-group ${RESOURCE_GROUP} \
  --gateway-name appgw \
  -n MyRule \
  --http-listener MyHttpListener \
  --rule-type Basic \
  --address-pool MyAddressPool --http-settings MyHttpSettings

```

## Create Virtual Machine Scale Set

Scale set is group of identical computers, typically app servers which have same configuration and application installed. In our case we have inside docker image which hosts on port 8080 sample web application. Traffic from internet is routed via load balance r with public IP.

```bash
# Create VMSS
az vmss create \
  --resource-group ${RESOURCE_GROUP} \
  --name vmss \
  --admin-username myadmin \
  --authentication-type ssh \
  --location ${LOCATION} \
  --image "OpenLogic:CentOS:7-CI:latest" \
  --vnet-name ${VNET} \
  --subnet ${SUBNETVMSS} \
  --vm-sku Standard_D1_v2 \
  --instance-count 2 \
  --ssh-key-value "${MYSSHKEY}" \
  --custom-data cloud-init-app.txt \
  --upgrade-policy-mode automatic

# Create Load Balancer probe
az network lb probe create \
  --resource-group ${RESOURCE_GROUP} \
  --name vmssLBProbe \
  --lb-name vmssLB \
  --protocol http --port 8080 --path /

# Create Load Balancer rule
az network lb rule create \
  --resource-group ${RESOURCE_GROUP} \
  --name vmssLBRule \
  --lb-name vmssLB \
  --backend-pool-name vmssLBBEPool \
  --backend-port 8080 \
  --frontend-ip-name loadBalancerFrontEnd \
  --frontend-port 80 \
  --protocol tcp \
  --probe-name vmssLBProbe
```