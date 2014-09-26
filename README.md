Crear multiples instancias en Zend Framework 2
==============


Zend Framework 2 crear 2 o más conexiones/instancias del adaptador DB adapter instances utilizando 'adaptadores'

Zend Framework 2.2 viene con abstract_factories que nos permite configurar varias instancias del adaptador DB con nombre. 

Zend\Db\Adapter\AdapterAbstractServiceFactory 


//config/autoload/global.php 

```
#!php
return array(
 'db' => array(
     	// esto es para para configurar el adaptador db principal .... 
        'driver' => 'Pdo',
        'dsn' => 'mysql:dbname=db_principal;host=localhost',
        'driver_options' => array(
            PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES \'UTF8\''
        ),
        // configurar otro adaptador cuando se necesita ... 
        'adapters' => array(
            'adptador1' => array(
                'driver' => 'Pdo',
                'dsn' => 'mysql:dbname=db_nueva;host=localhost',
                'driver_options' => array(
                    PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES \'UTF8\''
                ),
            ),
            'adptador2' => array(
                'driver' => 'Pdo',
                'dsn' => 'mysql:dbname=db_segunda;host=localhost',
                'driver_options' => array(
                    PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES \'UTF8\''
                ),
            ),
        )
    ),
    'service_manager' => array(  
    	// Para llamar al adaptador db principal 
        'factories' => array(
            'Zend\Db\Adapter\Adapter'
                    => 'Zend\Db\Adapter\AdapterServiceFactory',
        ),
        // Para permitir que otro adaptador pueda ser llamado 
        'abstract_factories' => array(
            'Zend\Db\Adapter\AdapterAbstractServiceFactory',
        )
    ),
);
```

   

//config/autoload/local.php 


```
#!php

return array(
    'db' => array(
    	// Usuario y contraseña en adaptador db principal
        'username' => 'root',
        'password' => '',
        'adapters' => array(
        	// Usuario y contraseña en adaptador db adptador1
            'adptador1' => array(
                'username' => 'root',
                'password' => '',
            ),
            // Usuario y contraseña en adaptador db adptador2
            'adptador2' => array(
                'username' => 'other_user',
                'password' => 'other_user_passwd',
            ),
        ),
    ),
);
```
poder instanciar ya solo resta configurar el archivo Module.php en el metodo getServiceConfig() 

Para probar si funciona puedes cambiar el actual por el que instancia a la base de datos alterno, ojo tu consulta debe ser sobre la base de datos nueva.

Obtener el adaptador principal **$dbAdapter = $sm->get('Zend\Db\Adapter\Adapter');**

Obtener cualquier adaptador que configuramos anteriormente: **$dbAdapter = $sm->get('adptador1');**


Utilizando el ejemplo que nos muestra Zend 2, tenemos la siguiente configuración:

Module.php


```
#!php

namespace Album;

 // Add these import statements:
 use Album\Model\Album;
 use Album\Model\AlbumTable;
 use Zend\Db\ResultSet\ResultSet;
 use Zend\Db\TableGateway\TableGateway;

 class Module
 {
     // getAutoloaderConfig() and getConfig() methods here

     // Add this method:
     public function getServiceConfig()
     {
         return array(
             'factories' => array(
                 'Album\Model\AlbumTable' =>  function($sm) {
                     $tableGateway = $sm->get('AlbumTableGateway');
                     $table = new AlbumTable($tableGateway);
                     return $table;
                 },
                 'AlbumTableGateway' => function ($sm) {
                     $dbAdapter = $sm->get('Zend\Db\Adapter\Adapter');
                     $resultSetPrototype = new ResultSet();
                     $resultSetPrototype->setArrayObjectPrototype(new Album());
                     return new TableGateway('album', $dbAdapter, null, $resultSetPrototype);
                 },
             ),
         );
     }
 }
```

En este caso para que puedan convivir las dos bases de datos y poder acceder a ellas es de la siguiente manera:

1. Agregamos los nuevos modelos a utilizar: 

```
#!php

use Album\Model\Otra;
use Album\Model\OtraTable;
```
1. Modificamos getServiceConfig() con la siguiente estructura:


```
#!php

public function getServiceConfig()
    {
        return array(
            'factories' => array(
               'Album\Model\AlbumTable' =>  function($sm) {
                     $tableGateway = $sm->get('AlbumTableGateway');
                     $table = new AlbumTable($tableGateway);
                     return $table;
                 },
                 'AlbumTableGateway' => function ($sm) {
                     $dbAdapter = $sm->get('Zend\Db\Adapter\Adapter');
                     $resultSetPrototype = new ResultSet();
                     $resultSetPrototype->setArrayObjectPrototype(new Album());
                     return new TableGateway('album', $dbAdapter, null, $resultSetPrototype);
                 },          
                'Album\Model\OtraTable' =>  function($sm) {
                    $tableGateway = $sm->get('OtraGateway');
                    $table = new OtraTable($tableGateway);
                    return $table;
                },
                 'OtraGateway' => function ($sm) {
                    $dbAdapter = $sm->get('adapatador1');
                    $resultSetPrototype = new ResultSet();
                    $resultSetPrototype->setArrayObjectPrototype(new Otra());
                    return new TableGateway('tabla', $dbAdapter, null, $resultSetPrototype);
                },
            ),
        );
    }
```
Listo, ya en tu controlador solo resta instanciar el adaptador


```
#!php

public function getOtraTable()
    {
        if (!$this->otraTable) {
            $sm = $this->getServiceLocator();
            $this->otraTable = $sm->get('Producto\Model\OtraTable');
        }
        return $this->otraTable;
    }
```
También debe agregar:


```
#!php

protected $otraTable;
```

Ahora podemos llamar getOtraTable() dentro de nuestro controlador cada vez que tenemos que interactuar con nuestro modelo de la base nueva.
