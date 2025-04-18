### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW

## Escalamiento en Azure con Maquinas Virtuales, Sacale Sets y Service Plans

### Dependencias
* Cree una cuenta gratuita dentro de Azure. Para hacerlo puede guiarse de esta [documentación](https://azure.microsoft.com/es-es/free/students/). Al hacerlo usted contará con $100 USD para gastar durante 12 meses.

### Parte 0 - Entendiendo el escenario de calidad

Adjunto a este laboratorio usted podrá encontrar una aplicación totalmente desarrollada que tiene como objetivo calcular el enésimo valor de la secuencia de Fibonnaci.

**Escalabilidad**
Cuando un conjunto de usuarios consulta un enésimo número (superior a 1000000) de la secuencia de Fibonacci de forma concurrente y el sistema se encuentra bajo condiciones normales de operación, todas las peticiones deben ser respondidas y el consumo de CPU del sistema no puede superar el 70%.

### Parte 1 - Escalabilidad vertical

1. Diríjase a el [Portal de Azure](https://portal.azure.com/) y a continuación cree una maquina virtual con las características básicas descritas en la imágen 1 y que corresponden a las siguientes:
    * Resource Group = SCALABILITY_LAB
    * Virtual machine name = VERTICAL-SCALABILITY
    * Image = Ubuntu Server 
    * Size = Standard B1ls
    * Username = scalability_lab
    * SSH publi key = Su llave ssh publica

![Imágen 1](images/part1/part1-vm-basic-config.png)

2. Para conectarse a la VM use el siguiente comando, donde las `x` las debe remplazar por la IP de su propia VM (Revise la sección "Connect" de la virtual machine creada para tener una guía más detallada).

    `ssh scalability_lab@xxx.xxx.xxx.xxx`

3. Instale node, para ello siga la sección *Installing Node.js and npm using NVM* que encontrará en este [enlace](https://linuxize.com/post/how-to-install-node-js-on-ubuntu-18.04/).
4. Para instalar la aplicación adjunta al Laboratorio, suba la carpeta `FibonacciApp` a un repositorio al cual tenga acceso y ejecute estos comandos dentro de la VM:

    `git clone <your_repo>`

    `cd <your_repo>/FibonacciApp`

    `npm install`

5. Para ejecutar la aplicación puede usar el comando `npm FibinacciApp.js`, sin embargo una vez pierda la conexión ssh la aplicación dejará de funcionar. Para evitar ese compartamiento usaremos *forever*. Ejecute los siguientes comando dentro de la VM.

    ` node FibonacciApp.js`

6. Antes de verificar si el endpoint funciona, en Azure vaya a la sección de *Networking* y cree una *Inbound port rule* tal como se muestra en la imágen. Para verificar que la aplicación funciona, use un browser y user el endpoint `http://xxx.xxx.xxx.xxx:3000/fibonacci/6`. La respuesta debe ser `The answer is 8`.

![](images/part1/part1-vm-3000InboudRule.png)

7. La función que calcula en enésimo número de la secuencia de Fibonacci está muy mal construido y consume bastante CPU para obtener la respuesta. Usando la consola del Browser documente los tiempos de respuesta para dicho endpoint usando los siguintes valores:
    * 1000000
    * 1010000
    * 1020000
    * 1030000
    * 1040000
    * 1050000
    * 1060000
    * 1070000
    * 1080000
    * 1090000    

8. Dírijase ahora a Azure y verifique el consumo de CPU para la VM. (Los resultados pueden tardar 5 minutos en aparecer).

![Imágen 2](images/part1/part1-vm-cpu.png)

9. Ahora usaremos Postman para simular una carga concurrente a nuestro sistema. Siga estos pasos.
    * Instale newman con el comando `npm install newman -g`. Para conocer más de Newman consulte el siguiente [enlace](https://learning.getpostman.com/docs/postman/collection-runs/command-line-integration-with-newman/).
    * Diríjase hasta la ruta `FibonacciApp/postman` en una maquina diferente a la VM.
    * Para el archivo `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` cambie el valor del parámetro `VM1` para que coincida con la IP de su VM.
    * Ejecute el siguiente comando.

    ```
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
    ```

10. La cantidad de CPU consumida es bastante grande y un conjunto considerable de peticiones concurrentes pueden hacer fallar nuestro servicio. Para solucionarlo usaremos una estrategia de Escalamiento Vertical. En Azure diríjase a la sección *size* y a continuación seleccione el tamaño `B2ms`.

![Imágen 3](images/part1/part1-vm-resize.png)

11. Una vez el cambio se vea reflejado, repita el paso 7, 8 y 9.
12. Evalue el escenario de calidad asociado al requerimiento no funcional de escalabilidad y concluya si usando este modelo de escalabilidad logramos cumplirlo.
13. Vuelva a dejar la VM en el tamaño inicial para evitar cobros adicionales.

**Preguntas**

1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?
2. ¿Brevemente describa para qué sirve cada recurso?
3. ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación que ejecutamos con el comando `npm FibonacciApp.js`? ¿Por qué debemos crear un *Inbound port rule* antes de acceder al servicio?
4. Adjunte tabla de tiempos e interprete por qué la función tarda tando tiempo.
5. Adjunte imágen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.
6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:
    * Tiempos de ejecución de cada petición.
    * Si hubo fallos documentelos y explique.
7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls` (no solo busque especificaciones de infraestructura)?
8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?, ¿Qué pasa con la FibonacciApp cuando cambiamos el tamaño de la VM?
9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?
10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?
11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?

### Parte 2 - Escalabilidad horizontal

#### Crear el Balanceador de Carga

Antes de continuar puede eliminar el grupo de recursos anterior para evitar gastos adicionales y realizar la actividad en un grupo de recursos totalmente limpio.

1. El Balanceador de Carga es un recurso fundamental para habilitar la escalabilidad horizontal de nuestro sistema, por eso en este paso cree un balanceador de carga dentro de Azure tal cual como se muestra en la imágen adjunta.

![](images/part2/part2-lb-create.png)

2. A continuación cree un *Backend Pool*, guiese con la siguiente imágen.

![](images/part2/part2-lb-bp-create.png)

3. A continuación cree un *Health Probe*, guiese con la siguiente imágen.

![](images/part2/part2-lb-hp-create.png)

4. A continuación cree un *Load Balancing Rule*, guiese con la siguiente imágen.

![](images/part2/part2-lb-lbr-create.png)

5. Cree una *Virtual Network* dentro del grupo de recursos, guiese con la siguiente imágen.

![](images/part2/part2-vn-create.png)

#### Crear las maquinas virtuales (Nodos)

Ahora vamos a crear 3 VMs (VM1, VM2 y VM3) con direcciones IP públicas standar en 3 diferentes zonas de disponibilidad. Después las agregaremos al balanceador de carga.

1. En la configuración básica de la VM guíese por la siguiente imágen. Es importante que se fije en la "Avaiability Zone", donde la VM1 será 1, la VM2 será 2 y la VM3 será 3.

![](images/part2/part2-vm-create1.png)

2. En la configuración de networking, verifique que se ha seleccionado la *Virtual Network*  y la *Subnet* creadas anteriormente. Adicionalmente asigne una IP pública y no olvide habilitar la redundancia de zona.

![](images/part2/part2-vm-create2.png)

3. Para el Network Security Group seleccione "avanzado" y realice la siguiente configuración. No olvide crear un *Inbound Rule*, en el cual habilite el tráfico por el puerto 3000. Cuando cree la VM2 y la VM3, no necesita volver a crear el *Network Security Group*, sino que puede seleccionar el anteriormente creado.

![](images/part2/part2-vm-create3.png)

4. Ahora asignaremos esta VM a nuestro balanceador de carga, para ello siga la configuración de la siguiente imágen.

![](images/part2/part2-vm-create4.png)

5. Finalmente debemos instalar la aplicación de Fibonacci en la VM. para ello puede ejecutar el conjunto de los siguientes comandos, cambiando el nombre de la VM por el correcto

```
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/vm1/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

Realice este proceso para las 3 VMs, por ahora lo haremos a mano una por una, sin embargo es importante que usted sepa que existen herramientas para aumatizar este proceso, entre ellas encontramos Azure Resource Manager, OsDisk Images, Terraform con Vagrant y Paker, Puppet, Ansible entre otras.

#### Probar el resultado final de nuestra infraestructura

1. Porsupuesto el endpoint de acceso a nuestro sistema será la IP pública del balanceador de carga, primero verifiquemos que los servicios básicos están funcionando, consuma los siguientes recursos:

```
http://52.155.223.248/
http://52.155.223.248/fibonacci/1
```

2. Realice las pruebas de carga con `newman` que se realizaron en la parte 1 y haga un informe comparativo donde contraste: tiempos de respuesta, cantidad de peticiones respondidas con éxito, costos de las 2 infraestrucruras, es decir, la que desarrollamos con balanceo de carga horizontal y la que se hizo con una maquina virtual escalada.

3. Agregue una 4 maquina virtual y realice las pruebas de newman, pero esta vez no lance 2 peticiones en paralelo, sino que incrementelo a 4. Haga un informe donde presente el comportamiento de la CPU de las 4 VM y explique porque la tasa de éxito de las peticiones aumento con este estilo de escalabilidad.

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Preguntas**

* ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?
* ¿Cuál es el propósito del *Backend Pool*?
* ¿Cuál es el propósito del *Health Probe*?
* ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?.
* ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?
* ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?
* ¿Cuál es el propósito del *Network Security Group*?
* Informe de newman 1 (Punto 2)
* Presente el Diagrama de Despliegue de la solución.


# SOLUCION

## IMAGENES
### 1.
![](images/img1.png)

### 2.
![](images/img2.png)

### 3.
![](images/img3.png)

### 4.
![](images/img4.png)

### 5.
![](images/img5.png)

### 6.
![](images/img6.png)

### 7.
![](images/img7.png)

### 8.
![](images/img8.png)

### 9.
![](images/img9.png)

### 10.
![](images/img10.png)

### 11.
![](images/img11.png)

### 12.
![](images/img12.png)

### 13.
![](images/img13.png)

### 14.
![](images/img14.png)

### 15.
![](images/img15.png)

### 16.
![](images/img16.png)



## PREGUNTAS 1 Y 2

### Preguntas 1
#### 1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?
Azure crea 6 recursos adicionales junto con la VM:  
* **Public IP address:** Dirección IP pública dedicada a la VM para comunicarse con Internet.  
* **Network Security Group:** Controla el tráfico de red entrante y saliente mediante reglas.  
* **Virtual Network:** Red privada que conecta recursos de Azure de forma segura.  
* **Network Interface:** Permite la comunicación de la VM con otros recursos y redes.  
* **SSH Key:** Clave utilizada para conectarse a la VM de manera segura.  
* **Disk:** Almacenamiento virtual gestionado por Azure, equivalente a un disco físico pero en la nube.

---

#### 2. ¿Brevemente describa para qué sirve cada recurso?
* **Public IP address:** Permite que la VM tenga comunicación con Internet y otros servicios públicos.  
* **Network Security Group:** Define reglas que controlan el tráfico de red que entra o sale de la VM.  
* **Virtual Network:** Actúa como una red privada en Azure para conectar recursos.  
* **Network Interface:** Conexión física/virtual que permite a la VM interactuar con redes externas.  
* **SSH Key:** Herramienta de autenticación para acceso remoto seguro.  
* **Disk:** Proporciona almacenamiento persistente para datos y el sistema operativo.

---

#### 3. ¿Por qué se cae la aplicación al cerrar la conexión SSH y por qué se necesita un *Inbound port rule*?
* La aplicación se cierra porque el proceso iniciado con `npm FibonacciApp.js` está ligado a la sesión SSH activa. Al cerrar la sesión, el proceso termina.  
* Es necesario un *Inbound port rule* para permitir el tráfico entrante al puerto que usa la aplicación (por ejemplo, el 3000), ya que por defecto Azure bloquea el tráfico.

---

#### 4. Adjunte tabla de tiempos e interprete por qué la función tarda tanto tiempo.
La función de Fibonacci tarda tanto tiempo porque calcula todos los números de la secuencia hasta el solicitado. El tiempo de ejecución crece exponencialmente con el número objetivo debido a la complejidad del cálculo.

---

#### 5. Adjunte imagen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.
El consumo de CPU es alto porque los cálculos de Fibonacci requieren muchos recursos computacionales. Cada iteración agrega más carga debido al número de operaciones que se ejecutan.

---

#### 6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:
* **Tiempos de ejecución de cada petición:**  
  Antes del escalamiento, las respuestas son más lentas debido a la limitación de recursos de la VM. Después del escalamiento, el tiempo mejora gracias al aumento de capacidad.  

* **Si hubo fallos, documéntelos y explique:**  
  El error ECONNRESET ocurre cuando la conexión se interrumpe inesperadamente. Esto puede ser causado por límites de la VM, reinicio de servicios o problemas de red.

---

#### 7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls`?
* **B2ms:** Tiene más memoria (2 GiB), mayor rendimiento de CPU base (40%) y mejor capacidad para manejar tráfico de red.  
* **B1ls:** Es más limitado, con 0.5 GiB de memoria y un rendimiento base de CPU del 10%. Está diseñado para cargas de trabajo ligeras.  

Ambos son económicos y se usan para tareas no críticas, pero `B2ms` soporta más carga y tráfico.

---

#### 8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?
No es una solución escalable. Si el número de peticiones sigue creciendo, incluso una VM más grande no será suficiente. Una mejor solución sería usar escalado horizontal (varias instancias).  
Al cambiar el tamaño de la VM, la aplicación deja de funcionar temporalmente y debe ser reiniciada.

---

#### 9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?
Cambiar el tamaño de la VM requiere reiniciarla, lo que provoca desconexión SSH y tiempo de inactividad. Esto puede afectar la disponibilidad del servicio.

---

#### 10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta?
Sí, hubo mejora porque al aumentar los recursos (CPU y memoria), la aplicación puede manejar más peticiones y reducir el tiempo de procesamiento.

---

#### 11. Aumente la cantidad de ejecuciones paralelas del comando de postman a 4. ¿El comportamiento del sistema es porcentualmente mejor?
Sí, el sistema muestra un mejor rendimiento proporcional porque aprovecha la capacidad adicional de CPU para procesar más solicitudes simultáneamente.

---

### Preguntas 2

#### 1. ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?

Azure ofrece dos tipos principales de balanceadores de carga:  
- **Internal Load Balancer (ILB):** Se utiliza para equilibrar la carga dentro de una red virtual privada. Es ideal para aplicaciones internas.  
- **Public Load Balancer:** Distribuye tráfico desde Internet hacia las máquinas virtuales en Azure.  

**SKU (Stock Keeping Unit):** En el contexto de Azure, define las capacidades y configuraciones del balanceador de carga.  
Tipos de SKU:  
- **Basic:** Ofrece características básicas, es más económica pero con menos capacidad y escalabilidad.  
- **Standard:** Ofrece características avanzadas como soporte para zonas de disponibilidad, mejor rendimiento y seguridad.  

El balanceador de carga necesita una IP pública para recibir solicitudes desde Internet y distribuirlas entre los recursos backend.

---

#### 2. ¿Cuál es el propósito del *Backend Pool*?

El *Backend Pool* es un conjunto de recursos (como máquinas virtuales) que reciben las solicitudes distribuidas por el balanceador de carga. Sirve para organizar y asociar los recursos que manejarán el tráfico.

---

#### 3. ¿Cuál es el propósito del *Health Probe*?

El *Health Probe* monitorea la salud de los recursos en el *Backend Pool*. Si un recurso no responde correctamente, el balanceador deja de enviarle tráfico hasta que vuelva a estar operativo.

---

#### 4. ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?

La *Load Balancing Rule* define cómo se distribuyen las solicitudes entre los recursos del *Backend Pool*. Configura el puerto de entrada, el puerto de destino y el protocolo.

Tipos de sesión persistente:  
- **None:** Cada solicitud puede ser manejada por cualquier recurso del backend.  
- **Client IP:** El tráfico del cliente se redirige siempre al mismo recurso, según su IP.  
- **Client IP and Protocol:** Igual que el anterior, pero también considera el protocolo.  

Esto es importante porque la persistencia afecta cómo las aplicaciones manejan sesiones y estados. Una alta dependencia de persistencia puede limitar la escalabilidad horizontal.

---

#### 5. ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?

- **Virtual Network (VNet):** Es una red privada en Azure que permite la comunicación segura entre recursos como máquinas virtuales y servicios.  
- **Subnet:** Una división lógica dentro de una VNet que segmenta el tráfico y organiza los recursos.  
- **Address Space:** Define el rango de direcciones IP asignadas a la VNet.  
- **Address Range:** Es el rango de IP dentro de una *Subnet*.  

Estos conceptos son fundamentales para el diseño y organización de la infraestructura en Azure.

---

#### 6. ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?

- **Availability Zone:** Son ubicaciones físicas únicas dentro de una región de Azure. Cada zona tiene centros de datos independientes con energía, refrigeración y redes propios.  
Seleccionar 3 zonas asegura alta disponibilidad y tolerancia a fallos.  

Una IP *zone-redundant* está diseñada para ser accesible incluso si una o más zonas fallan, asegurando confiabilidad.

---

#### 7. ¿Cuál es el propósito del *Network Security Group*?

El *Network Security Group (NSG)* actúa como un firewall que filtra el tráfico hacia y desde los recursos de Azure. Permite crear reglas de seguridad que controlan el acceso basado en el origen, destino, protocolo y puertos.

#### Presente el Diagrama de Despliegue de la solución.
![](images/img17.png)

