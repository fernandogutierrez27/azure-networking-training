# azure-networking-training

El siguiente documento tiene por objetivo llevar a cabo algunas pruebas de concepto relativas a Networking en Azure. Entre los puntos abordados destacan:
1. Creación de Vnet
2. Creación de Subnet
3. Peering
4. Implementación de transitividad entre Vnets a través de la implementación de un NVA.
5. Configuración de Network Security Groups

![image](https://user-images.githubusercontent.com/17756717/178268692-0e7286dd-5f8c-4a1e-8d7d-61df6648dd0d.png)

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

4. Luego la VM con el rol de NVA para la vnet Hub:
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

5. Finalmente, la VM en la red de la oficina sur:
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

2. Para guardar la IP xxx.xxx.xxx.xxx en la variable `norteip` y luego la IP yyy.yyy.yyy.yyy en la variable `nvaip`, ejecutamos:
```
norteip=xxx.xxx.xxx.xxx
nvaip=yyy.yyy.yyy.yyynvai
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
> Nota: Para diferencias los comandos ejecutados desde una VM a los ejecutados desde la Azure Cloud Shell, hemos agregado el nombre de la VM al comando. Para poder ejecutar el comando anterior con éxito, no se debe copiar la parte que dice `azureuser@of-norte-001-vm:~$`, esto es solo para referenciar el origen del comando.

5. Ambos ping deberían fallar.

## Paso 7: Generación de peering entre `of-norte-vnet` y `hub-vnet` a través del portal
A continuación, genaremos el peering entre oficina norte y el hub a través del portal de Azure.

1. Ingresamos a `oficina-norte-vnets` desde el listado de recursos:

![image](https://user-images.githubusercontent.com/17756717/178075786-fee703ec-fabc-4344-bf1d-1e9b63541ddc.png)

2. Seleccionamos `Peerings` (bajo la sección Settings) en menú lateral izquierdo:

![image](https://user-images.githubusercontent.com/17756717/178075890-1389597c-7163-47c9-9af8-3e4c4a567f87.png)

3. A continuación, presionamos en el botón `Add`:

![image](https://user-images.githubusercontent.com/17756717/178075929-06eb0651-020c-45a9-bbdf-140564b5a094.png)

4. Bajo la sección de This virtual network, en Peering link name ingresamos como nombre `oficina-norte-vnet-to-hub-vnet`, luego, en la sección Remote virtual network ingresamos como Peering link name el valor `hub-vnet-to-oficina-norte-vnet`.

![image](https://user-images.githubusercontent.com/17756717/178076198-1856b98f-7767-40ee-9c75-81950d2596c7.png)

5. Seleccionamos `hub-vnet` en el selector de Virtual network de la sección Remote virtual network. Finalmente, mantenemos todos los valores predeterminados y presionamos sobre el botón `Add`:

![image](https://user-images.githubusercontent.com/17756717/178076368-51cfe71a-1139-499f-b155-bdb430d0f02f.png)

6. A continuación, deberíamos poder ver el nuevo Peering, con el estado Connected. Si el estado es Updating, por favor espere unos minutos y vuelva a recargar la vista.

![image](https://user-images.githubusercontent.com/17756717/178076472-73fc97e1-71c5-46e5-a23e-6ae7582ca58b.png)

7. Además, si revisamos la sección Peering de la `hub-vnet`, encontraremos que una nueva conexión de Peering se ha creado automáticamente:

![image](https://user-images.githubusercontent.com/17756717/178076576-2a691af4-990d-4d9b-97c4-e52c034b6c3c.png)

8. Finalmente realizamos una prueba de ping desde `of-norte-001-vm` hacia `nva-001-vm`, la cual debería ser exitosa. Con esto validamos que la reciente conexión de Peering ha funcionado correctamente.
```
azureuser@of-norte-001-vm:~$ ping 172.16.1.4 -c 4 -W 1
```

## Paso 8: Generación de Vnet Peering entre `oficina-sur-vnet` y `hub-vnet`
A continuación, generaremos el Peering para las redes faltantes a través de Azure CLI.

1. Generamos Peering `hub-vnet-to-oficina-sur-vnet`
```
az network vnet peering create \
    --name hub-vnet-to-oficina-sur-vnet \
    --remote-vnet oficina-sur-vnet \
    --resource-group networking-training-rg \
    --vnet-name hub-vnet \
    --allow-vnet-access \
    --allow-forwarded-traffic
```

2. Luego realizamos la conexión en el sentido contrario:
```
az network vnet peering create \
    --name oficina-sur-vnet-to-hub-vnet \
    --remote-vnet hub-vnet \
    --resource-group networking-training-rg \
    --vnet-name oficina-sur-vnet \
    --allow-vnet-access \
    --allow-forwarded-traffic
```

3. A continuación, comprobamos que al ingresar a `nva-001-vm`, es posible llegar a `of-norte-001-vm` y `of-sur-001-vm`
```
ssh azureuser@$nvaip

azureuser@nva-001-vm:~$ ping 10.0.1.4 -c 4 -W 1
azureuser@nva-001-vm:~$ ping 192.168.1.4 -c 4 -W 1
```

4. No obstante, si ingresamos mediante ssh a `of-norte-001-vm` y realizamos un ping hace `of-sur-001-vm`, el comando va a fallar. Es decir, la conexión de Peering no es transitiva!!

## Paso 9: Habilitación de tráfico transitivo en hub
Para permitir el tráfico transitivo a través del NVA, debemos permitir IP forwarding tanto en la NIC de la VM (en Azure), como dentro del sistema operativo (Ubuntu)

1. Ejecutamos el siguiente comando para habilitar IP forwarding en la NIC de `nva-001-vm`
```
az network nic update --name nva-001-vmVMNic -g networking-training-rg --ip-forwarding true
```
> Nota: Es posible que el nombre entregado en el parámetro `--name` sea diferente, dependiendo del nombre utilizado para crear la VM. La manera más sencilla de validar el nombre de la NIC, es revisando en el listado del grupo de recursos:
> ![image](https://user-images.githubusercontent.com/17756717/178077823-1177382c-489f-4355-9d61-e9fe507a5b92.png) 

2. Luego nos conectamos a la `nva-001-vm` y habilitamos ipforwarding dentro de ella:
```
ssh azureuser@$nvaip

azureuser@nva-001-vm:~$ sudo vim /etc/sysctl.conf
```

3. Descomentamos la línea que dice `net.ipv4.ipforward=1`, cerramos Vim (`ESC` + `:wq!` + `ENTER`) y luego ejecutamos el siguiente comando para persistir los cambios:
```
azureuser@nva-001-vm:~$ sudo sysctl -p
```

## Paso 10: Configuración de tabla de ruta en `it-norte-subnet` para utilización de NVA
Es esta sección, vamos a configurar una User Defined Route para `it-norte-subnet`, que enrutará todo el tráfico destinado a `192.168.1.0/24` hacia nuestro `nva-001-vm`.

1. Ingresamos a `oficina-norte-vnet`, luego vamos a la sección de Subnets y seleccionamos `it-norte-subnet`. Finalmente, validamos que no existe ninguna Route table seleccionada.

![image](https://user-images.githubusercontent.com/17756717/178081768-cbf003f1-4806-4829-98a3-5261a2d22e97.png)

2. Posteriormente, escribímos `route` en la barra de búsqueda superior y seleccioanmos Route tables.

![image](https://user-images.githubusercontent.com/17756717/178081826-6fba1ded-2bd1-449c-b0f6-d68505a4cd8e.png)

3. Pulsamos en el botón `Create`.

4. Seleccionamos nuestro grupo de recursos (`network-training-rg`), la región en la que estamos trabajando (`East US 2`) y agregamos como nombre `it-norte-udr`. Finalmente presionamos el botón `Review + create` y en la página siguiente presionamos en `Create`.

![image](https://user-images.githubusercontent.com/17756717/178081942-2c004c2e-684c-4ec1-8de3-25d070c957b2.png)

5. Una vez creado el recurso, accedemos a éste. Luego vamos a la sección `Routes` del menú de la izquierda y presionamos sobre el botón `Add`:

![image](https://user-images.githubusercontent.com/17756717/178082036-c25e3c70-acff-4439-a961-98b25ab6e099.png)

7. Agregamos la siguiente configuración y luego presionamos en el botón `Add`:
* Route name: `it-sur-subnet-to-nva`
* Address prefix destination: `IP Addresses`
* Destination IP addresses/CIDR ranges: `192.168.1.0/24`
* Next hop type: `Virtual appliance`
* Next hop address: `172.16.1.4`

![image](https://user-images.githubusercontent.com/17756717/178082182-6c36c347-785f-496f-be66-f2801d3ea3e8.png)

8. Posteriormente, vamos a la sección subnets y hacemos click en `Associate`. Luego seleccionamos la Virtual network `oficina-norte-vnet` y la subnet `it-norte-subnet`. Finalmente, presionamos el botón `OK`.

![image](https://user-images.githubusercontent.com/17756717/178082291-7a5e7d46-5d7a-458f-ad81-7ca160175b64.png)

9. Volvemos a ingresar a la Subnet (`it-norte-subnet`) y veremos que ahora tenemos asociada la Route table `it-norte-udr`:

![image](https://user-images.githubusercontent.com/17756717/178082351-ea6df19b-7230-4909-86af-376f2570d013.png)

10. Finalmente, es necesario repetir los pasos anteriores, para crear una User defined route para la subnet `it-sur-subnet`, tomando en cuenta la siguiente configuración:
* Nombre de Route table: `it-sur-udr`.
* Ruta a agregar:
  * Route name: `it-norte-subnet-to-nva`
  * Address prefix destination: `IP Addresses`
  * Destination IP addresses/CIDR ranges: `10.0.1.4/24`
  * Next hop type: `Virtual appliance`
  * Next hop address: `172.16.1.4`

## Paso 11: Ping final
Finalmente, ingresamos a `of-norte-001-vm` y ejecutamos un ping hacia `192.168.1.4`
```
azureuser@of-norte-001-vm:~$ ping 192.168.1.4 -c 4 -W 1
```
Si el ping ha funcionado correctamente, **Felicitaciones!!**

## Bonus track: NSG
Tarea:
* Crear un NSG y asociarlo a la Subnet `dmz-subnet`.
* Denegar todo el tráfico entrando que se dirija a `it-sur-subnet`.
* Probar ping desde `of-norte-001-vm` hacia `of-sur-001-vm`: Debe fallar.
* Agregar regla a NSG que permita tráfico desde `it-norte-subnet` hacia `it-sur-subnet`.
* Probar nuevamente ping, ahora debe funcionar.
