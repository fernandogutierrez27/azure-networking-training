# azure-networking-training

0. crear grupo de recursos
* Vaya a la barra de búsqueda superior y escriba `resource groups`, luego el elemento del listado como aparece en la imagen:
![image](https://user-images.githubusercontent.com/17756717/178022031-dbf729e1-727f-42dd-94c5-c43f2bb708de.png)

* Presione en create, arriba a la izquierda
![image](https://user-images.githubusercontent.com/17756717/178022127-44ba672b-eb19-45e9-8ab8-4410838b041b.png)

* Ingrese `azure-network-training-rg` como nombre del RG, seleccione la región `East US 2` y presione en `Review + create`. Finalmente, presione create en la panatalla siguiente.
![image](https://user-images.githubusercontent.com/17756717/178022277-7df44bb5-77e1-4e7c-873e-655cf719063a.png)


1. crear 3 vnets

* Acceda al grupo de recursos recientemente creado y haga click sobre el botón `Create` (arriba a la izquierda).
* Escriba `virtual network`sobre el cuadro de búsqueda y presione enter.
* Seleccione el primer elemento del listado y haga click en `Create`
* Seleccione el grupo de recursos creado con anterioridad y la región a desplegar la virtual network. Finalmente agregue un nombre a la vnet y presiones `Next: IP Addresses`
* Elimine el IPv4 address space predeterminado
* Ingrese el siguiente CIDR para definir el Address Space: 10.0.0.0/8
* Presione sobre el botón Add subnet
* Ingrese `public-subnet` como nombre de la subnet y el siguiente Subnet address range `10.0.1.0/24`. Finalmente presione el botón add.
* Presione en `Review + create`y finalmente en `create`
------------------------
* Presione en `Next: Security`
* En Security, habilite la opción BastionHost, ingrese nombre del bastion y `10.0.0.0/24` como AzureBastionSubnet address space.
* En public IP address seleccione create new e ingrese como nombre `bastion-pip`
* Mantengan las otras opciones predeterminadas y pulse en Review + create
------------------------

A continuación, crearemos 2 vnets adicionales, `dmz-vnet` y `cliente-vnet`, además, para cada uno crearemos una subnet, `dmz-subnet` y `ntt-subnet` respectivamente.
### Habilitación de Azure CLI
* Haga click en el ícono de consola ubicado arriba a la derecha, esto iniciará una Azure Cloud CLI.
* Seleccione Bash como lenguage y luego presione en el botón `Create storage`
* Presione en show advanced settings
* En Resource group, seleccione Use existing y luego seleccione el grupo de recursos creado para el tutorial.
* En Storage Account, seleccione create new e ingrese un nombre cuenta de Storage Account, Azure validará que este nombre sea único y solo contenga números y letras.
* En File share, seleccione Create new e ingrese un nombre de File share
* Ejecute el siguiente comando para generar la vnet y subnet de DMZ:
```
az network vnet create \
    --resource-group networking-training-rg \
    --name dmz-vnet \
    --address-prefixes 172.16.0.0/12 \
    --subnet-name dmz-subnet \
    --subnet-prefixes 172.16.1.0/24 \
    --location eastus2
```
* Ejecute el siguiente comando para generar la vnet y subnet de cliente:
```
az network vnet create \
    --resource-group networking-training-rg \
    --name cliente-vnet \
    --address-prefixes 192.168.0.0/16 \
    --subnet-name ntt-subnet \
    --subnet-prefixes 192.168.1.0/24 \
    --location eastus2
```

2. Crear 3 VMs (una en cada vnet con su respectivo bastion)
* Open the Cloud Shell editor and create a file named cloud-init.txt.
```
code cloud-init.txt
```

* agregar siguiente information al archivo. con esta configuración inetutils-traceroute es instalado, lo cual contiene la utilidad de traceroute
```
#cloud-config
package_upgrade: true
packages:
   - inetutils-traceroute
```

* Generar la VM para la red ntt
```
az vm create \
    --resource-group networking-training-rg \
    --name ntt-vm \
    --vnet-name ntt-vnet \
    --subnet public-subnet \
    --image UbuntuLTS \
    --admin-username azureuser \
    --size Standard_B1s \
    --generate-ssh-keys \
    --custom-data cloud-init.txt
```

* Luego la VM con el rol de NVA para la DMZ
```
az vm create \
    --resource-group networking-training-rg \
    --name dmz-vm \
    --vnet-name dmz-vnet \
    --subnet dmz-subnet \
    --image UbuntuLTS \
    --admin-username azureuser \
    --size Standard_B1s \
    --generate-ssh-keys \
    --custom-data cloud-init.txt
```

* Finalmente, la VM en la red del cliente
```
az vm create \
    --resource-group networking-training-rg \
    --name cliente-vm \
    --vnet-name cliente-vnet \
    --subnet ntt-subnet \
    --image UbuntuLTS \
    --admin-username azureuser \
    --size Standard_B1s \
    --generate-ssh-keys \
    --custom-data cloud-init.txt
```

* Acceder a vm NTT
```
# suggested IP: 10.0.1.4
ssh azureuser@<ntt-vm-public-ip>
```

* realizar ping a DMZ y cliente
```
ping <dmz-vm-private-ip> -c 4 -W 1 # suggested IP 172.16.1.4
ping <cliente-vm-private-ip> -c 4 -W 1 #suggested IP 192.168.1.4
```


3. generar peering entre 1-2, 2-3
- usar portal
* Ir a ntt-vnet
* Seleccionar peerings en menú a la izquierda)
* Seleccionar Add
* Ingresar nombre `ntt-vnet-to-dmz-vnet`
* Ingresar nombre remoto `dmz-vnet-to-ntt-vnet`
* seleccionar dmz-vnet como vnet remota
* Validar estado en ambas vnets

* Validar ping

- Generar peering para dmz-cliente
* En Azure CLI ingresar
```
az network vnet peering create \
    --name dmz-vnet-to-cliente-vnet \
    --remote-vnet cliente-vnet \
    --resource-group networking-training-rg \
    --vnet-name dmz-vnet \
    --allow-vnet-access \
    --allow-forwarded-traffic
```
* Luego, realizar la conexión inversa
```
az network vnet peering create \
    --name cliente-vnet-to-dmz-vnet \
    --remote-vnet dmz-vnet \
    --resource-group networking-training-rg \
    --vnet-name cliente-vnet \
    --allow-vnet-access \
    --allow-forwarded-traffic
```

4. comprobar conectividad entre 1 y 2, 3 y 2, pero no entre 1 y 3
* Probar conexión desde DMZ a ambos destinos
* Luego probar conexión desde ntt-vm a cliente-vm --> Fallo

5. habilitar tráfico en nva (DMZ)
* actualizamos la nic para habilitar IP forwarding en la NIC de la dmz-vim
```
az network nic update --name dmz-vmVMNic -g networking-training-rg --ip-forwarding true
```

* habilitamos ipforwarding dentro de la dmz-vm
```
sudo vim /etc/sysctl.conf
```
* descomentar net.ipv4.ipforward=1 y luego ejecutar el siguiente comando para actualizar:
```
sudo sysctl -p
```
* ir a ntt-vnet > public-subnet, validar que no hay tabla asociada
* buscar route y seleccionar route tables
* click en Create
* completar resource group, region y nombre (ntt-udr)
* click en Review + create y luego en Create
* ir al recurso

* ir a Routes y luego dar click en Add
* agregar nombre: cliente-route
* seleccionar address prefix destination: IP addresses
* ingresar prefijo de red: ntt-subnet (en cliente-vnet) 192.168.1.0/24
* seleccionar Next hop type: virtual appliance
* ingresar Next hop address: dmz-vm internal ip (172.16.1.4)

* Ir a la sección subnets y hacer click en associate
* seleccionar ntt-vnet
* Seleccionar ntt-subnet
* la UDR se ha asociado a dmz-subnet

* Ingresar a subnet y validar route table



6. probar conectividad entre 1 y 3
```
ping 192.168.1.4
```

7. agregar NSG en 3
* Crear NSG
* Asociar a subnet
* Denegar todo el tráfico desde todos los origenes

8. validar desconexión
* Probar ping desde ntt-vm
``` Azure CLI
azureuser@dmz-vm:~$ ping 192.168.1.4
```
* Agregar regla a NSG que permite tráfico entrante desde 10.0.1.0/24
* Probar ping desde ntt-vm
* Probar ping desde dmz-vm
