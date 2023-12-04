# DynamoDB

---

# Introducción

Este documento se centra en proporcionar una guía detallada para la instalación y configuración en sistemas Ubuntu, facilitando así la implementación de DynamoDB.

La guía aborda desde la instalación de Python y las bibliotecas necesarias hasta la configuración de la AWS CLI para gestionar las credenciales de acceso. Además, se incluye un ejemplo de código en Python utilizando Boto3, el SDK de AWS, para interactuar con DynamoDB.

El imforme incluye un ejemplo práctico de Python que utiliza la biblioteca mencionada anteriormente para interactuar con DynamoDB. La clase 'Reservations' proporciona funcionalidades clave, como la creación de tablas, la implementación de reservas, la carga de datos de muestra y la lectura de reservas, permitiendo una implementación rápida y eficiente de la lógica de reserva mediante DynamoDB.

---

# Plan de implementación

## Instalación de Bibliotecas Necesarias

### Paso 1: Instalar Python y pip
Es necesario tener Python y pip instalados en el sistema Ubuntu. Se pueden instalar con los siguientes comandos:

```bash
sudo apt update
sudo apt install python3
sudo apt install python3-pip
```

### Paso 2: Instalar Boto3 y PrettyTable
Boto3 es el SDK de AWS para Python, necesario para interactuar con los servicios de AWS, incluido DynamoDB. PrettyTable es una biblioteca que ayuda a mostrar datos en formato de tabla. Ambos se pueden instalar mediante pip:

```bash
pip install boto3
pip install prettytable
```


## Configuración de AWS CLI

### Paso 3: Instalar AWS CLI
La AWS Command Line Interface (CLI) es esencial para configurar las credenciales de AWS y administrar los servicios de AWS desde la línea de comandos. Se instala con:

```bash
sudo apt install awscli
```

### Paso 4: Configurar AWS CLI
Para configurar la AWS CLI, se ejecuta el siguiente comando y se siguen las instrucciones para ingresar las credenciales de AWS (ID de clave de acceso y clave de acceso secreta):

```bash
aws configure
```


Se solicitará:
- AWS Access Key ID: Ingresar el AWS Access Key ID.
- AWS Secret Access Key: Ingresar el AWS Secret Access Key.
- Default region name: Elegir la región, como 'us-east-1'.
- Default output format: Dejar en blanco o poner 'json'.

## Código Python para Interactuar con DynamoDB

La clase `Reservations` encapsula las funcionalidades para interactuar con una tabla de DynamoDB destinada a almacenar datos de reservaciones. Esta clase incluye métodos para:

- Inicializar la clase con recursos de DynamoDB y una tabla existente.
- Crear una tabla en DynamoDB con claves de partición y ordenamiento específicas.
- Generar fechas aleatorias para las reservaciones.
- Añadir reservaciones a la tabla.
- Cargar un número específico de reservaciones como datos de muestra.
- Leer un número limitado de reservaciones usando un escaneo completo de la tabla.
- Leer reservaciones filtradas por el número de mesa.
- Eliminar todas las reservaciones asociadas a un número de mesa específico.

---

# Implementación

El uso práctico de esta clase incluye la inicialización con un recurso de DynamoDB y el nombre de una tabla, seguido de la ejecución de diversas operaciones como la creación de la tabla, la carga de datos de muestra, la lectura y la eliminación de reservaciones.

```python
import boto3
from boto3.dynamodb.conditions import Attr
from botocore.exceptions import ClientError
import logging
import random
from datetime import datetime, timedelta
from prettytable import PrettyTable

class Reservations:
    """Encapsulates an Amazon DynamoDB table of reservation data."""

    def __init__(self, dyn_resource, table_name):
        """
        :param dyn_resource: A Boto3 DynamoDB resource.
        :param table_name: The name of the existing DynamoDB table.
        """
        self.dyn_resource = dyn_resource
        self.table = self.dyn_resource.Table(table_name)

    def create_table(self, table_name):
        """
        Creates an Amazon DynamoDB table that can be used to store reservation data.
        The table uses the reservation ID as the partition key and the date-time as the sort key.

        :param table_name: The name of the table to create.
        :return: The newly created table.
        """
        try:
            self.table = self.dyn_resource.create_table(
                TableName=table_name,
                KeySchema=[
                    {"AttributeName": "reservaID", "KeyType": "HASH"},  # Partition key
                    {"AttributeName": "fechaHora", "KeyType": "RANGE"},  # Sort key
                ],
                AttributeDefinitions=[
                    {"AttributeName": "reservaID", "AttributeType": "S"},
                    {"AttributeName": "fechaHora", "AttributeType": "S"},
                ],
                ProvisionedThroughput={
                    "ReadCapacityUnits": 5,
                    "WriteCapacityUnits": 5,
                },
            )
            self.table.wait_until_exists()
        except ClientError as err:
            logging.error(
                "Couldn't create table %s. Here's why: %s: %s",
                table_name,
                err.response["Error"]["Code"],
                err.response["Error"]["Message"],
            )
            raise
        else:
            return self.table

    def random_date(self):
        """
        Generates a random date for the reservation.
        """
        start_date = datetime.now()
        end_date = start_date + timedelta(days=365)
        random_date = start_date + (end_date - start_date) * random.random()
        return random_date.strftime("%Y-%m-%d %H:%M:%S")

    def add_reservation(self, reservaID, fechaHora, nombreCliente, numeroMesa):
        """
        Adds a reservation to the table.

        :param reservaID: The ID of the reservation.
        :param fechaHora: The date and time of the reservation.
        :param nombreCliente: The name of the client making the reservation.
        :param numeroMesa: The table number for the reservation.
        """
        try:
            self.table.put_item(
                Item={
                    "reservaID": reservaID,
                    "fechaHora": fechaHora,
                    "nombreCliente": nombreCliente,
                    "numeroMesa": numeroMesa,
                }
            )
        except ClientError as err:
            logging.error(
                "Couldn't add reservation %s to table %s. Here's why: %s: %s",
                reservaID,
                self.table.name,
                err.response["Error"]["Code"],
                err.response["Error"]["Message"],
            )
            raise

    def load_sample_data(self, num_reservations=500):
        """
        Loads a specified number of sample reservations into the table.

        :param num_reservations: The number of sample reservations to load.
        """
        for i in range(num_reservations):
            reserva_id = f"resv-{i}"
            fecha_hora = self.random_date()
            nombre_cliente = f"Cliente-{random.randint(1, 1000)}"
            numero_mesa = random.randint(1, 15)
            self.add_reservation(reserva_id, fecha_hora, nombre_cliente, numero_mesa)

    def read_reservations(self, limit=10):
        """
        Reads a limited number of reservations from the table using scan.

        :param limit: The maximum number of reservations to read.
        :return: A list of reservation items.
        """
        try:
            response = self.table.scan(Limit=limit)
            return response.get('Items', [])
        except ClientError as err:
            logging.error(
                "Couldn't read reservations from table %s. Here's why: %s: %s",
                self.table.name,
                err.response["Error"]["Code"],
                err.response["Error"]["Message"],
            )
            raise

    def read_reservations_by_table(self, table_number, limit=10):
        """
        Reads reservations for a specific table number using scan with a filter.

        :param table_number: The table number to filter reservations.
        :param limit: The maximum number of reservations to read.
        :return: A list of reservation items.
        """
        try:
            response = self.table.scan(
                FilterExpression=Attr('numeroMesa').eq(table_number),
                Limit=limit
            )
            return response.get('Items', [])
        except ClientError as err:
            logging.error(
                "Couldn't read reservations for table number %s from table %s. Here's why: %s: %s",
                table_number, self.table.name,
                err.response["Error"]["Code"],
                err.response["Error"]["Message"],
            )
            raise

    def delete_reservations_by_table(self, table_number):
        """
        Deletes all reservations for a specific table number.

        :param table_number: The table number for which to delete reservations.
        """
        try:
            # Primero, encontrar todas las reservas para esa mesa
            response = self.table.scan(
                FilterExpression=Attr('numeroMesa').eq(table_number)
            )
            reservations = response.get('Items', [])

            # Luego, borrar cada reserva encontrada
            for reservation in reservations:
                self.table.delete_item(
                    Key={
                        'reservaID': reservation['reservaID'],
                        'fechaHora': reservation['fechaHora']
                    }
                )
            print(f"Deleted all reservations for table number {table_number}.")
        except ClientError as err:
            logging.error(
                "Error deleting reservations for table number %s. Here's why: %s: %s",
                table_number,
                err.response["Error"]["Code"],
                err.response["Error"]["Message"],
            )
            raise


# Example usage
dyn_resource = boto3.resource('dynamodb', region_name='sa-east-1')
table_name = 'Reservations'  # Nombre de tu tabla existente en DynamoDB
reservations = Reservations(dyn_resource, table_name)
reservations_table = reservations.create_table(table_name)
print(f"Created table {reservations_table.name}")

# Load sample data
reservations.load_sample_data()

# Read and print a few reservations
sample_reservations = reservations.read_reservations(limit=10)
print("Reservations:")

sample_reservations.sort(key=lambda x: x['fechaHora'])

# Create a PrettyTable object
table = PrettyTable()
table.field_names = ["Reserva ID", "Fecha y Hora", "Nombre Cliente", "Número Mesa"]

# Add rows to the table
for reservation in sample_reservations:
    table.add_row([
        reservation['reservaID'],
        reservation['fechaHora'],
        reservation['nombreCliente'],
        int(reservation['numeroMesa'])
    ])

# Print the table
print(table)

# Read and print reservations for table number 7
table_number = 7
reservations_by_table = reservations.read_reservations_by_table(table_number, limit=500)

# Create a PrettyTable object
table = PrettyTable()
table.field_names = ["Reserva ID", "Fecha y Hora", "Nombre Cliente", "Número Mesa"]

# Add rows to the table
for reservation in reservations_by_table:
    table.add_row([
        reservation['reservaID'],
        reservation['fechaHora'],
        reservation['nombreCliente'],
        int(reservation['numeroMesa'])  # Convert Decimal to int
    ])

# Print the table
print(table)

# Borrar todas las reservas para la mesa número 7
table_number = 7
reservations.delete_reservations_by_table(table_number)

# Read and print reservations for table number 7
table_number = 7
reservations_by_table = reservations.read_reservations_by_table(table_number, limit=500)

# Create a PrettyTable object
table = PrettyTable()
table.field_names = ["Reserva ID", "Fecha y Hora", "Nombre Cliente", "Número Mesa"]

# Add rows to the table
for reservation in reservations_by_table:
    table.add_row([
        reservation['reservaID'],
        reservation['fechaHora'],
        reservation['nombreCliente'],
        int(reservation['numeroMesa'])  # Convert Decimal to int
    ])

# Print the table
print(table)
```
---

# Resultados y conclusiones

![image](https://github.com/ItiKvnAlf/BD-columnar/assets/103331361/b9d2f58a-0801-46ae-9eeb-7db7ccc92b4d)

En la primera imagen se muestra el resultado de la creación de la tabla 'Reservations', imprimiendo para cada una de las filas su identificación, fecha y hora en la que se realizó la reserva, el nombre del cliente y el número de mesa.

![image](https://github.com/ItiKvnAlf/BD-columnar/assets/103331361/4640ac33-e761-4927-9b39-1d076c14e355)

Luego, se buscan e imprimen todas las reservas que tengan como número de mesa el valor '10', para después eliminar todas las reservas que hagan referencia al mismo. La tabla resultante se muestra vacía.

La implementación explicada a detalle durante este informe demuestra cómo DynamoDB puede ser utilizado de manera eficaz para gestionar datos de reservas, proporcionando un rendimiento rápido, escalabilidad y flexibilidad en el modelo de datos. La importancia de DynamoDB está en su capacidad para manejar eficientemente operaciones de lectura, escritura y consulta en entornos donde la velocidad y la escalabilidad son críticas. Este enfoque de base de datos NoSQL es especialmente valioso en comparación con otras APIs debido a su rendimiento optimizado y a la naturaleza totalmente administrada del servicio, y además poder trabajar junto a otros servicios proporcionados por AWS.

---
# Referencias

- [Amazon DynamoDB Documentation] https://docs.aws.amazon.com/dynamodb/
- [Boto3 documentation] https://boto3.amazonaws.com/v1/documentation/api/latest/index.html
- [prettytable 3.9.0] https://pypi.org/project/prettytable/
