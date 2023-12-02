# Wordpress บน Azure Container Instant

## รายละเอียด

![diagram](./diagram/acidemo%20diagram-Wordpress%20on%20ACI%20Example.drawio.png)

ตัวอย่างนี้จะนำเอา Azure Container Instant (ACI) มาใช้ในการ Run Wordpress โดยจะมีการใช้ Azure Files เป็น Persistant Storage ให้กับ Container และ Azure Database for MySQL - Flexible Server (PaaS) เป็น Database ทั้งนี่้มีการนำ Azure Load Balancer และ ACI จำนวน 2 Instant เพื่อเพิ่ม High Available ให้กับระบบโดยรวมด้วย

โดยจะใช้งาน Container Image จาก [bitnami/wordpress](https://hub.docker.com/r/bitnami/wordpress) (สามารถกดตาม [Links](https://hub.docker.com/r/bitnami/wordpress) เพื่อไปศึกษา Parameter ที่ใช้ในการ run เพิ่มเติมได้)

ตัวอย่างนี้จะใช้ [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/) เป็นเครื่องมือในการ Deployment โดยสามารถใช้งานผ่าน Cloud Shell หรือ สามารถ download ไปติดตั้งจาก <https://learn.microsoft.com/en-us/cli/azure/install-azure-cli>

## ขั้นตอน

### การเตรียม Resource ที่เกี่ยวข้อง

1. ประกาศตัวแปรที่จะใช้ สามารถปรับแต่งได้ตามความเหมาะสม

   ```sh
   resourcegroup="container-rg"
   location="eastasia"
   vnetname="demo-vnet"
   zonename="wordpress.private.mysql.database.azure.com"
   storageacct="aciwp$RANDOM"
   mysqldb="wpdb$RANDOM"
   mysqldbfqdn="$mysqldb.$zonename"
   mysqldbuser="wpdbuser"
   mysqldbpasswd="Pa55w.rd1234"
   ```

2. สร้าง Resource Group

   ```sh
   az group create --resource-group $resourcegroup --location $location
   ```

3. สร้าง Azure Virtual Network สำหรับ Deployment

   ```sh
   az network vnet create --resource-group $resourcegroup --location $location --name $vnetname --address-prefixes '10.122.0.0/16'
   az network vnet subnet create --resource-group $resourcegroup --vnet-name $vnetname --name 'aci-subnet' --address-prefixes '10.122.0.0/24' --delegations 'Microsoft.ContainerInstance/containerGroups' --service-endpoints 'Microsoft.Storage' 
   az network vnet subnet create --resource-group $resourcegroup --vnet-name $vnetname --name 'mysql-subnet' --address-prefixes '10.122.1.0/24' --delegations 'Microsoft.DBforMySQL/flexibleServers'
   acisubnetid=$(az network vnet subnet show --resource-group $resourcegroup --vnet-name $vnetname --name 'aci-subnet' --query "id" --output tsv)
   mysqlsubnetid=$(az network vnet subnet show --resource-group $resourcegroup --vnet-name $vnetname --name 'mysql-subnet' --query "id" --output tsv)
   ```

4. สร้าง Private DNS Zone

   ```sh
   az network private-dns zone create --resource-group $resourcegroup --name $zonename
   az network private-dns link vnet create --resource-group $resourcegroup --name vnetlink --zone-name $zonename --virtual-network $vnetname --registration-enabled true
   dnszoneid=$(az network private-dns zone show --resource-group $resourcegroup --name $zonename --query "id" --output tsv)
   ```

5. สร้าง Storage Account สำหรับเป็น Persistant Storage

   ```sh
   az storage account check-name --name $storageacct
   az storage account create --resource-group $resourcegroup --location $location --name $storageacct --sku Standard_LRS
   storageacctkey=$(az storage account keys list --account-name $storageacct --resource-group $resourcegroup --query "[?contains(keyName,'key1')].value" --output tsv)
   az storage share create --account-name $storageacct --account-key $storageacctkey --name wwwroot
   az storage share create --account-name $storageacct --account-key $storageacctkey --name cert
   ```

6. สร้าง Azure Database for MySQL Flexible Server
   สามารถเปลี่ยน Parameter --admin-user และ --admin-password ตามที่เราต้องการ

   ```sh
   az mysql flexible-server create --resource-group $resourcegroup --location $location --name $mysqldb --sku-name Standard_B1ms --tier Burstable --storage-size 20 --version 8.0.21 --subnet $mysqlsubnetid --admin-user $mysqldbuser --admin-password $mysqldbpasswd --private-dns-zone $dnszoneid
   az mysql flexible-server parameter set --resource-group $resourcegroup --server-name $mysqldb --name 'require_secure_transport' --value 'OFF'
   az mysql flexible-server db create --resource-group $resourcegroup --server-name $mysqldb --database-name wordpress
   ```

### การ Run Azure Container Instant

Container Image [bitnami/wordpress](https://hub.docker.com/r/bitnami/wordpress) สามารถเปิด HTTPS จาก Container ได้เลยโดยเราสามารถนำ Server Certificate ไปไว้ที่ PATH /cert ของ Container ได้เลย ซึ่งในตัวอย่างนี้เราจะใช้ Azure Files พาท cert ที่เราสร้างไว้ก่อนหน้า Map เข้าไปใน Container ดัังนั้นเราสามารถ Upload Server Certificate ไปเก็บไว้ที่ fileshare cert ได้เลย (server.crt สำหรับ publickey, server.key สำหรับ private key และ server-ca.crt สำหรับ CA Chain ถ้ามี )

1. Upload Serever Certificate (เปลี่ยนไฟล์ **[server_com.*]** เป็นชื่อไฟล์ SSL Certificate ของเรา)

   ```sh
   az storage file upload --account-name $storageacct --account-key $storageacctkey --share-name cert --source server_com.crt --path server.crt
   az storage file upload --account-name $storageacct --account-key $storageacctkey --share-name cert --source server_com.key --path server.key
   az storage file upload --account-name $storageacct --account-key $storageacctkey --share-name cert --source server_com.ca-bundle --path server-ca.crt
   ```

2. สร้าง Container Instant ทีละตัว โดยรอให้ตัวแรกทำการ ติดตั้ง Wordpress Script และ Initial Database เรียบร้อยก่อนแล้วจึงรันตัวที่เหลือตาม โดยการรันครั้งนี้จะใช้ YAML ไฟล์ โดยใช้ Template จากไฟล์ [wordpress.yaml](./wordpress.yaml) โดยแก้ไขค่าที่อยู่ใน ```<$xxxx>``` ให้ตรงกับที่เราสร้างไว้ก่อนหน้า (หากทำตามขั้นตอนสามารถเช็คได้โดย echo ตัวแปลนั้นๆ เช่น ```<$location>``` ก็ให้ใส่ค่าที่ได้จากการ ```echo $location``` เป็นต้น) 

   ```yaml
   apiVersion: '2021-10-01'
   location: <$location>
   name: aciwp1
   properties:
      containers:
      - name: wordpress
         properties:
            image: bitnami/wordpress
            environmentVariables:
            - name: WORDPRESS_DATABASE_HOST
               value: <$mysqldbfqdn>
            - name: WORDPRESS_DATABASE_USER
               secureValue: <$mysqldbuser> 
            - name: WORDPRESS_DATABASE_PASSWORD
               secureValue: <$mysqldbpasswd>
            - name: WORDPRESS_DATABASE_NAME
               value: wordpress
            - name: WORDPRESS_USERNAME
               secureValue: wp_admin
            - name: WORDPRESS_PASSWORD
               secureValue: wp_password
            - name: WORDPRESS_BLOG_NAME
               value: WordpressBLOG
            - name: WORDPRESS_FIRST_NAME
               secureValue: wp_admin_firstname
            - name: WORDPRESS_LAST_NAME
               secureValue: wp_user_lastname
            - name: WORDPRESS_ENABLE_HTTPS
               value: true
            ports:
            - port: 8080
               protocol: TCP
            - port: 8443
               protocol: TCP
            resources:
               requests:
                  cpu: 1.0
                  memoryInGB: 2.0
            volumeMounts:
            - mountPath: /bitnami/wordpress
               name: wwwvolume
            - mountPath: /certs
               name: certvolume
      restartPolicy: Always
      osType: Linux
      subnetIds:
         - id: <$acisubnetid>
      volumes:
      - name: wwwvolume
         azureFile:
            sharename: wwwroot
            storageAccountName: <$storageacct>
            storageAccountKey: <$storageacctkey>
      - name: certvolume
         azureFile:
            sharename: cert
            storageAccountName: <$storageacct>
            storageAccountKey: <$storageacctkey>
   tags: {}
   type: Microsoft.ContainerInstance/containerGroups
   ```

   *สำหรับค่า Parameter เพิ่มเติมของ bitnami/wordpress สามารถศึกษาได้จาก <https://hub.docker.com/r/bitnami/wordpress>*

3. ทำการ Deploy Container Instant ด้วย yaml ไฟล์ในขั้นตอนที่แล้ว

   ```sh
   az container create --resource-group $resourcegroup --file aciwp1.yaml
   ```

4. ตรวจสอบสถานะการ Run Bootstrap script ของ Container ด้วยการดู Log ของ Azure Container Instant

   ```sh
   az container logs --resource-group $resourcegroup --name aciwp1 --follow
   ```

   รอจนกว่าจะเจอข้อความ ```** WordPress setup finished! **``` แล้ว กด ```<ctrl> + c``` เพื่อออกจากหน้า Logs

5. สร้างไฟล์ yaml อีกไฟล์ เหมือนขั้นตอนที่ 2 แต่เปลี่ยน ```name: aciwp1``` เป็น ```name: aciwp2``` และทำการสร้าง Container Instant อีก 1 Instant

   ```sh
   az container create --resource-group $resourcegroup --file aciwp2.yaml
   ```

6. Azure Loadbalancer เพื่อทำหน้าที่ Loadbalance

   ```sh
   az network public-ip create --resource-group $resourcegroup --location $location --name loadbalance_pip --sku Standard --zone 1 2 3
   loadbalanceIP=$(az network public-ip show --resource-group $resourcegroup --name loadbalance_pip --query 'ipAddress' --output tsv)
   az network lb create --resource-group $resourcegroup --location $location --name wplb --sku Standard --public-ip-address loadbalance_pip --frontend-ip-name default-fe --backend-pool-name aci-be
   az network lb probe create --resource-group $resourcegroup --lb-name wplb --name wphttpprobe --protocol Http --port 8080 --path /
   az network lb rule create --resource-group $resourcegroup --lb-name wplb --name wphttps --protocol tcp --frontend-ip-name default-fe --frontend-port 443 --backend-pool-name aci-be --backend-port 8443 --probe-name wphttpprobe --disable-outbound-snat true
   ```

7. เพิ่ม ACI ที่สร้างเข้าไปใน Backend Pool ของ AzureLoadBalancer

   ```sh
   aciwplist=$(az container list --resource-group $resourcegroup --query '[].ipAddress.ip' --output tsv)
   for aci in $aciwplist
   do
      az network lb address-pool address add --resource-group $resourcegroup --lb-name wplb --pool-name aci-be --name "wp$aci" --ip-address $aci --subnet $acisubnetid
   done
   ```

8. Add Record บน DNS Server ของเรา ให้ชี้มาที่ IP ของ Load Balancer สามารถตรวจสอบ Public IP ได้ด้วย command ```az network public-ip show --resource-group $resourcegroup --name loadbalance_pip --query 'ipAddress' --output tsv```
9. สามารถเข้าใช้งาน Wordpress จาก Public IP ของ Azure Load Balancer หรือ FQDN ที่เรา Set ไว้ก็ได้ สำหรับ admin user, admin password ที่ใช้บริหารจัดการก็จะเป็นไปตามค่าที่ตั้งไว้ใน yaml ไฟล์หัวข้อ ```WORDPRESS_USERNAME``` และ ```WORDPRESS_PASSWORD``` หากไม่ได้เปลี่ยนค่า default username = wp_admin , password = wp_password