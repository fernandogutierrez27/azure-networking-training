# azure-networking-training

El siguiente documento tiene por objetivo llevar a cabo algunas pruebas de concepto relativas a Networking en Azure. Entre los puntos abordados destacan:
1. Creación de Vnet
2. Creación de Subnet
3. Peering
4. Implementación de transitividad entre Vnets a través de la implementación de un NVA.
5. Configuración de Network Security Groups


## Paso 1: Creación de Grupo de Recursos
Para comenzar, vamos a crear un grupo de recursos donde desplegaremos cada uno de los componentes de nuestra solución. Siga estos pasos para la creación del RG:
1. Vaya a la barra de búsqueda superior y escriba `resource groups`, luego seleccione el elemento del listado según aparece en la imagen:

![image](https://user-images.githubusercontent.com/17756717/178022031-dbf729e1-727f-42dd-94c5-c43f2bb708de.png)

2. Presione el botón `Create`, ubicado en el sector superior izquierdo.

![image](https://user-images.githubusercontent.com/17756717/178022127-44ba672b-eb19-45e9-8ab8-4410838b041b.png)

3. Ingrese `azure-network-training-rg` como nombre del RG, seleccione la región `East US 2` y presione en `Review + create`.

![image](https://user-images.githubusercontent.com/17756717/178022277-7df44bb5-77e1-4e7c-873e-655cf719063a.png)

4. Finalmente, en la pantalla siguiente presione el botón `Create`.

## Paso 2: Creación de Vnet mediante el portal.
1. A continuación, el grupo recientemente creado debería aparecer en el listado de grupos de recursos, haga click sobre éste para acceder:

![image](https://user-images.githubusercontent.com/17756717/178070199-df2647ba-db14-43fe-8cc0-435a160be76a.png)

2. Haga click sobre el botón `Create`, ubicado en el sector superior izquierdo de la pantalla.

![image](https://user-images.githubusercontent.com/17756717/178070316-70c1d7c6-dca1-4337-b644-a147bcc41e62.png)

3. Escriba `virtual network` sobre el cuadro de búsqueda y presione enter.

4. Seleccione el primer elemento del listado y luego presione sobre el botón `Create`.

![image](https://user-images.githubusercontent.com/17756717/178070449-c511fbd5-0307-415a-ae4a-639c30b5bbef.png)

5. Seleccione el grupo de recursos creado con anterioridad y la región a desplegar la Virtual Network (`East US 2`). Finalmente agregue como nombre de la Vnet `oficina-norte-vnet` y presione `Next: IP Addresses`.

![image](https://user-images.githubusercontent.com/17756717/178073503-1ac19588-7bab-4c7b-bb7a-2ded60f1c788.png)

6. En la siguiente pantalla, elimine el `IPv4 address space` predeterminado.

7. Ingrese el siguiente CIDR para definir el IPv4 address space: `10.0.0.0/8`

8. Presione sobre el botón `Add subnet`

9. Ingrese `it-norte-subnet` como nombre de la Subnet y el siguiente Subnet address range `10.0.1.0/24`. Finalmente presione el botón Add.

![image](https://user-images.githubusercontent.com/17756717/178073715-c020f7a3-7c3e-43e9-bfd5-1313a0d2df8f.png)

10. Presione en `Review + create`y finalmente en `Create`.

![image](https://user-images.githubusercontent.com/17756717/178073853-192ad9be-521e-48e0-a4ac-390e7de1a191.png)

## Paso 3: Habilitación de Azure CLI
Procederemos a habilitar la Azure Cloud Shell. Esta herramienta corresponde a una interfaz de línea de comandos que viene incorporada en el portal de Azure, la cual nos permitirá ejecutar diferentes comandos de Azure CLI.

1. Haga click en el ícono de consola ubicado arriba a la derecha, esto iniciará una Azure Cloud Shell.

![image](https://user-images.githubusercontent.com/17756717/178071763-d9c9df1a-5ef7-440b-82a9-631c2f565ab2.png)

2. Seleccione `Bash` como lenguage y luego presione en `Show advanced settings`.

3. Seleccione la región elegida con anterioridad (East US 2) o alguna de su preferencia.

4. En Resource group, seleccione `Use existing` y luego seleccione el grupo de recursos creado para el tutorial (networking-training-rg).

5. En Storage Account, seleccione `Create new` e ingrese un nombre cuenta de Storage Account, Azure validará que este nombre sea único y solo contenga números y letras.

6. En File share, seleccione `Create new` e ingrese un nombre de File share (ej: `azcli`).

7. Finalmente presione el botón `Create Storage`. Azure Cloud Shell hace uso de un recurso de File share dentro de un Storage account, para simular el sistema de archivos. Como resultado, ahora podrá empezar a escribir comando en Azure Cloud Shell.

![image](https://user-images.githubusercontent.com/17756717/178072276-2f905943-1021-4b60-8da0-6d560a85bb15.png)


## Paso 4: Creación de Vnets adicionales
A continuación, crearemos 2 Vnets adicionales llamadas `hub-vnet` y `oficina-sur-vnet`, además crearemos una Subnet para cada una de las Vnets, las cuales se llamarán `dmz-subnet` y `it-sur-subnet` respectivamente.

1. Ejecute el siguiente comando para generar la Vnet y Subnet de `hub`:
```
az network vnet create \
    --resource-group networking-training-rg \
    --name hub-vnet \
    --address-prefixes 172.16.0.0/12 \
    --subnet-name dmz-subnet \
    --subnet-prefixes 172.16.1.0/24 \
    --location eastus2
```

2. Ejecute el siguiente comando para generar la Vnet y Subnet de `oficina-sur`:
```
az network vnet create \
    --resource-group networking-training-rg \
    --name oficina-sur-vnet \
    --address-prefixes 192.168.0.0/16 \
    --subnet-name it-sur-subnet \
    --subnet-prefixes 192.168.1.0/24 \
    --location eastus2
```
3. Debería visualizarse los siguientes recursos en el resource gruop:

![image](https://user-images.githubusercontent.com/17756717/178074114-2eb7c61b-331e-455c-bc33-4c3e81290d32.png)

## Paso 5: Creación de Virtual Machines
A continuación, vamos a crear una VM en cada una de las Vnets, para poder realizar pruebas de conectividad.

1. En primer lugar, vamos a crear un archivo llamado cloud-init.txt, este archivo nos permitirá precargar algunas herramientas en nuestras VMs.
```
code cloud-init.txt
```

2. Agregar siguiente information al archivo. Con esta configuración `inetutils-traceroute` es instalado, lo cual contiene la utilidad de `traceroute`. Finalmente, presione `CTRL + S` para guardar y luego `CTRL + Q` para salir. 
```
#cloud-config
package_upgrade: true
packages:
   - inetutils-traceroute
```

3. Generamos la primera VM para la red de la oficina norte:
```
az vm create \
    --resource-group networking-training-rg \
    --name of-norte-001-vm \
    --vnet-name oficina-norte-vnet \
    --subnet it-norte-subnet \
    --image UbuntuLTS \
    --admin-username azureuser \
    --size Standard_B1s \
    --generate-ssh-keys \
    --custom-data cloud-init.txt \
    --no-wait
```

* Luego la VM con el rol de NVA para la vnet Hub:
```
az vm create \
    --resource-group networking-training-rg \
    --name nva-001-vm \
    --vnet-name hub-vnet \
    --subnet dmz-subnet \
    --image UbuntuLTS \
    --admin-username azureuser \
    --size Standard_B1s \
    --generate-ssh-keys \
    --custom-data cloud-init.txt \
    --no-wait
```

* Finalmente, la VM en la red de la oficina sur:
```
az vm create \
    --resource-group networking-training-rg \
    --name of-sur-001-vm \
    --vnet-name oficina-sur-vnet \
    --subnet it-sur-subnet \
    --image UbuntuLTS \
    --admin-username azureuser \
    --size Standard_B1s \
    --generate-ssh-keys \
    --custom-data cloud-init.txt
```
## Paso 6: Comprobación de acceso inicial
A continuación, validaremos que no tenemos acceso entre las diferentes redes.

1. Tomamos nota de cada una de las IPs públicas generadas para cada VM. En nuestro caso, almacenaremos el valor de la IP pública de `of-norte-001-vm` en la variable `$norteip` y de `nva-001-vm` en la variable `$nvaip`. Para listar las IPs públicas generadas, se debe utilizar el siguiente comando:
```
az vm list-ip-addresses --resource-group networking-training-rg --query '[].virtualMachine.network.publicIpAddresses'
```

2. Para guardar la IP xxx.xxx.xxx.xxx en la variable `norteip`, ejecutamos:
```
norteip=xxx.xxx.xxx.xxx
```

3. Accedemos a la VM `of-norte-001-vm`
```
ssh azureuser@$norteip
```

4. Realizamos ping a `nva-001-vm` y `of-sur-001-vm`, cuyas IPs deberían ser `172.16.1.4` y `192.168.1.4` respectivamente.
```
azureuser@of-norte-001-vm:~$ ping 172.16.1.4 -c 4 -W 1
azureuser@of-norte-001-vm:~$ ping 192.168.1.4 -c 4 -W 1
```
> Nota: Para diferencias los comandos ejecutados desde una VM a los ejecutados desde la Azure Cloud Shell, agregaremos el nombre de la VM en caso de aplicar. Para poder ejecutar el comando anterior con éxito, no se debe copiar la parte que dice `azureuser@of-norte-001-vm:~$`, esto es solo para referenciar el origen del comando.

5. Ambos ping deberían fallar.

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
```Azure CLI
azureuser@dmz-vm:~$ ping 192.168.1.4
```
* Agregar regla a NSG que permite tráfico entrante desde 10.0.1.0/24
* Probar ping desde ntt-vm
* Probar ping desde dmz-vm

