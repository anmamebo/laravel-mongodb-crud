# Laravel MongoDB CRUD

## Paso 1: Crear proyecto laravel

Si usas este repositorio, ya tendrás creado el proyecto, en caso de no tener el proyecto creado usar:
```
composer create-project laravel/laravel nombre-proyecto "9.*"
```

## Paso 2: Docker

Crear carpeta *docker* dentro de la estructura del proyecto de laravel, dentro de esta carpeta crear el archivo *Dockerfile* y el archivo *apache2.conf*, y en la carpeta base del proyecto laravel crear fichero *docker-compose.yml*, quedando una estructura tal que así:

![image](https://github.com/anmamebo/laravel-mongodb-crud/assets/74211239/8461c676-8c7a-4772-9919-854f41c0f69b)

En el archivo *Dockerfile*:
```
# Usamos la imagen base de PHP con Apache
FROM php:8.1-apache

# Establecemos el directorio de trabajo en /var/www/html
WORKDIR /var/www/html

# Actualizamos el índice de paquetes y luego instalamos las dependencias necesarias
RUN apt-get update && apt-get install -y \
    libzip-dev \
    unzip \
    && docker-php-ext-install zip \
    && docker-php-ext-install pdo_mysql

# Copiamos los archivos de la aplicación a la imagen del contenedor
COPY . /var/www/html

# Instalamos las dependencias de Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Instalamos las dependencias de Laravel
# RUN composer install --no-scripts --no-autoloader

# Generamos el archivo autoload de Composer
RUN composer dump-autoload

# Copiamos el archivo de configuración de Apache
COPY ./docker/apache2.conf /etc/apache2/sites-available/000-default.conf

# Habilitamos el módulo de reescritura de Apache
RUN a2enmod rewrite

# Exponemos el puerto 80 para acceder a la aplicación web
EXPOSE 80

RUN apt-get install -y libcurl4-openssl-dev pkg-config libssl-dev

# Instalamos las dependencias de MongoDB
RUN pecl install mongodb \
    && docker-php-ext-enable mongodb

RUN pecl config-set php_ini /etc/php.ini

# Ejecutamos el servidor Apache en primer plano
CMD ["apache2-foreground"]
```

En el archivo apache2.conf:
```
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /var/www/html/public

    <Directory /var/www/html/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

En el archivo docker-compose.yml:
```
version: '3'
services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    ports:
      - 8000:80
    environment:
    - DB_CONNECTION=mongodb
    - DB_HOST=mongodb
    - DB_PORT=27017
    - DB_DATABASE=mydatabase
    - DB_USERNAME=root
    - DB_PASSWORD=root
    volumes:
      - .:/var/www/html
    depends_on:
    - mongodb
  mongodb:
    image: mongo
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_DATABASE=mydatabase
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=root
  mongo-express:
    image: mongo-express
    ports:
      - 8081:8081
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=root
      - ME_CONFIG_MONGODB_ADMINPASSWORD=root
      - ME_CONFIG_MONGODB_URL=mongodb://root:root@mongodb:27017/
    depends_on:
    - mongodb
```

A continuación, ejecutar docker-compose up para iniciar los contenedores:

```
docker-compose up
```

## Paso 3: Instalación de paquetes para Mongo

Ejecutar el siguiente comando para saber los contenedores que están en ejecución y coger el id del contenedor *app*:

```
docker ps
```

Entrar en la consola del contenedor *app*:

```
docker exec -it <id_contenedor> bash
```

Instalar los paquetes:

```
composer require mongodb/mongodb
```

```
composer require jenssegers/mongodb
```

## Paso 4: Configuración fichero .env

Sustituir los valores del fichero .env con los siguientes:

```
DB_CONNECTION=mongodb
DB_HOST=127.0.0.1
DB_PORT=27017
DB_DATABASE=mydatabase
DB_USERNAME=root
DB_PASSWORD=root
```

## Paso 5: Configuración fichero database.php

Añadir en el fichero config/database.php dentro del array de *connections* la conexión con mongodb:

```
'mongodb' => [
            'driver' => 'mongodb',
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', 27017),
            'database' => env('DB_DATABASE'),
            'username' => env('DB_USERNAME'),
            'password' => env('DB_PASSWORD'),
            'options' => [
                // Opcional: Configuraciones adicionales de MongoDB
            ],
        ],
```

## Paso 6: Configuración fichero app.php

Añadir en el fichero config/app.php dentro del array de *providers*:

```
Jenssegers\Mongodb\MongodbServiceProvider::class,
```


## Paso 7: Ejecutar las migraciones

Desde dentro de la consola del contenedor *app*

```
php artisan migrate
```
Podemos acceder a *http://localhost:8081/* y comprobar que se ha creado la base de datos con las migraciones:

![image](https://github.com/anmamebo/laravel-mongodb-crud/assets/74211239/dcdc3ecc-cfd9-45fb-996b-69d9737e6146)
![image](https://github.com/anmamebo/laravel-mongodb-crud/assets/74211239/c66ff603-60c9-4758-a9d7-9e5f316432f8)

