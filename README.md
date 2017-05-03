### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW
## Ejercicio - Cachés y Bases de datos NoSQL

En este ejercicio se va a ajustar la aplicación de dibujo colaborativo, y se le va a agregar un caché para llevar de forma centralizada -y con un acceso rápido- el estado de los dibujos de cada una de las salas.


## Parte I

1. Inicie la máquina virtual Ubuntu trabajada anteriormente, e instale el servidor REDIS [siguiendo estas instrucciones](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-redis), sólo hasta 'sudo make install'. Con esto, puede iniciar el servidor con 'redis-server'. Nota: para poder hacer 'copy/paste' en la terminal (la de virtualbox no lo permite), haga una conexión ssh desde la máquina real hacia la virtual.

2. Agregue las dependencias requeridas para usar Jedis, un cliente Java para REDIS:

	```xml
		<dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.8.2</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.25</version>
        </dependency>   
 	```                               


        
2. Revise [en la documentación de REDIS](http://redis.io/topics/data-types) el tipo de dato HASH, y la manera como se agregan tablas hash a una determianda llave. Con esto presente, inicie un cliente redis (redis-cli) en su máquina virtual, y cree varias llaves, cada una asociada a cuatro tuplas (llave-valor) correspondientes: la palabra a adivinar, lo que se ha adivinado de la palabra hasta ahora, el ganador, y si la partida ha finalizado. Use como llave de los HASH una cadena compuesta, que incluya el identificador de la partida, por ejemplo: "partida:12345", "partida:9999", etc.    


4. Copie la siguiente clase y archivo de configuración (en las rutas respectivas) dentro de su proyecto (éstas ya tiene la configuración para manejar un pool de conexiones al REDIS):

	* [https://github.com/hcadavid/jedis-examples/blob/master/src/main/java/util/JedisUtil.java](https://github.com/hcadavid/jedis-examples/blob/master/src/main/java/util/JedisUtil.java)
	* [https://github.com/hcadavid/jedis-examples/blob/master/src/main/resources/jedis.properties](https://github.com/hcadavid/jedis-examples/blob/master/src/main/resources/jedis.properties)
   


## Parte II



En la versión actual de la aplicación, en el método dentro del servidor que recibe los eventos, se tiene una lógica que considera -dentro de un bloque sincronizado-:

1. Recibir el nuevo punto.
2. Agregar el punto a una colección de puntos.
3. Publicar el punto en el tópico correspondiente.
4. Si la colección tiene cuatro puntos:
	* Armar un polígono.
	* Publicar el polígono.
	* Reiniciar la colección.

El esquema anterior sin emabargo, SÓLO sirve cuando se tiene un único servidor. Cuando se tienen N bajo un esquema de balanceo de carga, evidentemente se pueden tener condiciones de carrera.

Para corregir esto, va a hacero uso de la base de datos Llave-valor REDIS, y su cliente correspondiente para Java Jedis:


```java

Jedis jedis = JedisUtil.getPool().getResource();
	    
	//Operaciones	    
	    
jedis.close();
	    
```


1. En lugar de tener una lista de puntos en la memoria del servidor, se tendrán dos listas de enteros: una para los valores en X y otra para los valores en Y.

2. El servidor, al recibir un nuevo punto:
	* Inicia [una transacción REDIS](https://github.com/xetorthio/jedis/wiki/AdvancedUsage), haciendo 'watch' sobre las dos listas anteriores.
	* Hace RPUSH a las dos listas, con los datos X y Y correspondientes ([ver API de Jedis](http://tool.oschina.net/uploads/apidocs/jedis-2.1.0/redis/clients/jedis/Jedis.html)).
	* Ejecuta la transacción (si tx es el objeto Transaction):
	
		```java
		List<Object> res=tx.exec();
		```	
	* Verifica si la transacción fue exitosa:la lista 'res' debería no ser vacía.
	* Si fue exitosa, publica el punto en el tópico correspondiente.
	* De lo contrario, reintentar la operación (puede hacer todo lo anterior dentro de un ciclo).
	* Cierra 

3. Una vez se haya agregado el punto en REDIS, se debe hacer una operación que, de manera atómica (esta vez sin transacción), consulte si hay 4 puntos en las listas, y que en caso afirmativo, devuelva dichos puntos y posteriormente los elimine.

	Para lo anterior, cree un [script en lenguaje LUA](https://www.redisgreen.net/blog/intro-to-lua-for-redis-programmers/), que:
	1. Invoque la operación LRANGE para consultar el tamaño de una de las listas.
	2.  En caso de que el mismo sea cuatro, guarde los cuatro elementos en una variable y borre la llave en REDIS. 
	3. Al final, el programa retorna una lista vacía si la lista de puntos NO tenía cuatro puntos, o la lista con dichos elementos en caso contrario. 
	
	El siguiente es el script correspondiente (ajuste los nombres de las llaves a su conveniencia), el cual puede ejecuatarse a través del método 'eval' (tanto de REDIS como de Jedis):

	```lua
	local out; 
	if (redis.call('LLEN','dalist')==4) then 
		out=redis.call('LRANGE','dalist',0,-1); 			
		redis.call('DEL','dalist'); 
		return out; 
	else 
		return {}; 
	end
	```
4. Si al ejecutarse el script LUA en REDIS se obtienen la lista de puntos, el servidor construye con los mismos el polígono, y los publica en el tópico correspondiente.


5. Una vez funcione la prueba de concepto, haga 'refactoring' para que la aplicación tenga claramente demarcadas las capas lógicas y de persistencia, y que se pueda cambiar del esquema de pesistencia en memoria al esquema de persistencia en REDIS. 


### Nota - Error de SockJS

En caso de que con la configuración planteada (aplicación y REDIS corriendo en la máquina virtual) haya conflictos con SockJS pruebe configurar REDIS para aceptar conexiones desde máquinas externas, editando el archivo /home/ubuntu/redis-stable/redis.conf, cambiando "bind 127.0.0.1" por "bind 0.0.0.0", y reiniciando el servidor con: 

	```bash
	redis-server /home/ubuntu/redis-stable/redis.conf. 
```
	Una vez hecho esto, en la aplicación ajustar el archivo jedis.properties, poner la IP de la máquina virtual (en lugar de 127.0.0.1), y ejecutarla desde el equipo real (en lugar del virtual). ** OJO: Esto sólo se hará como prueba de concepto!, siempre se le debe configurar la seguridad a REDIS antes de permitirle el acceso remoto!. **



### Parte IV - Para el Martes, Impreso. 


Actualizar (y corregir) el diagrama realizado en el laboratorio anterior.
