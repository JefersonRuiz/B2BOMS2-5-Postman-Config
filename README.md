# CONFIGURACIÓN POSTMAN

En este directorio se entregan dos archivos, uno correspondiente a la plantilla o *template* para el entorno en Postman (<code>SF Environment Template.postman_environment.json</code>), donde se almacenarán los datos de configuración para la conexión con Salesforce, y otro con la colección de los diferentes llamados o solicitudes que se realizan para validar el correcto funcionamiento y configuración de las pruebas (<code>SB2BOMS2-5 - Creación de Ofertas.postman_collection.json</code>).

También se ha creado un flujo en PostMan para tratar de automatizar la búsqueda de los registros asociados, dada la naturaleza del modelo de datos, y dando la posibilidad de crear multiples registros a la vez si llegara a ser necesario. Dicho flujo se puede ver y usar desde este enlace:

https://interstellar-meadow-845353.postman.co/workspace/COPEC-SF-Integration~a313bdc3-06a1-4574-858c-97203969e469/flow/66a3b13177a445003d6abf34

Si se desea utilizar este flujo, será necesario tener una cuenta en Postman y estar logueado en la plataforma, para poder luego hacer un *fork* del flujo.
## Pre-requisitos

Para poder conectarse correctamente por medio de las APIs de Salesforce, sean estas aquellas ya provistas por Salesforce, o creadas especificamente para un entorno determinado, es necesario contar con lo siguiente:

* Un usuario activo en el entorno de Salesforce, con permisos suficientes para poder hacer llamados hacia Salesforce.

* Una aplicación conectada configurada con los permisos necesarios, y que el usuario tenga a su asignado el uso de dicha aplicación conectada.

Para la integración se está utilizando inicialmente la aplicación conectada **AccountResAPI**.

## Plantilla de entorno

La plantilla de entorno se establecen las variables de entorno necesarias que las solicitudes funcionen correctamente, y de esta manera poder autorizar y conectar con entornos de Salesforce. Se recomienda duplicar el template entregado para la conexión con un nuevo entorno de Salesforce, dandole un nombre adecuada que identifique facilmente a qué entorno de Salesforce corresponde. Dichas variables son las siguientes:

* **Variables asociadas a la aplicación conectada (*Connected App*)**:
    * <code>sf_client_id</code>: *Hash* correspondiente al identificador de la aplicación conectada.
    * <code>sf_client_secret</code>: *Hash* correspondiente al secreto asociado a la aplicación conectada.

    Estos valores pueden ser los mismos para todos los usuarios de una misma organización, pero serán diferentes para cada una de las organizaciones de Salesforce, aún si se usa la misma aplicación conectada.

* **Variables asociadas al usuario de Salesforce**:
    * <code>sf_user_name</code>: Usuario del Salesforce.
    * <code>sf_password</code>: Contraseña del usuario de Salesforce.
    * <code>sf_user_token</code>: Token de seguridad asociado al usuario de Salesforce.

    Normalmente el usuario y su respectiva contraseña se entregan o configuran durante la creación del usuario. El token de seguridad será necesario generarlo la primera vez dentro de la configuración del usuario dentro de la plataforma de Salesforce.

* **Variables Asociadas al entorno de Salesforce**
    * <code>instance_url</code>: Corresponde a que tipo de organización se está tratando de acceder. Esta variable sera igual a <code>https://login.salesforce.com</code> si el entorno es uno productivo o *developer edition*, o será <code>https://test.salesforce.com</code> si el entorno es un *sandbox*.
    * <code>sf_url</code>: URL de la versión *classic* del entorno de Salesforce (https://\<<my-instance\>>.my.salesforce.com).
    * <code>access_token</code>: Token de autorización generado al realizar el llamado de autenticacíon y autorización (más adelante se ve este proceso).
    * <code>version</code>: Versión de API de Salesforce usada.

Si se necesita, siempre es posible agregar más variables de acuerdo a las necesidades y preferencias de cada usuario.

## Colección de llamados.

La colección de llamadas <code>B2BOMS2-5 - Creación de Ofertas.postman_collection</code> está enfocada en obtener los registros necesarios para la creación de las ofertas usando SOQL. En total son 7 solicitudes almanenadas, además de la necesaria para tener los permisos necesarios para interactuar con la API de Salesforce.

Esta colección utiliza las siguientes variables de colección:

* <code>current_account_id</code>: Correspondiente al Id de Salesforce de la cuenta asociada al Solicitante de prueba.
* <code>current_buyer_group_id</code>: Correspondiente al Id de Salesforce del registro `BuyerGroup` del cual la cuenta asociada al solicitante está asociada.
* <code>current_policy_id</code>: Asociada al registro `CommerceEntitlementPolicy` que está asociado al grupo de compradores al que pertenece la cuenta asociada al solicitante.
* <code>current_parent_product_id</code>: Asociada al registro `Product2` que es el producto padre de la oferta que se pretende crear.
* <code>current_child_product_id</code>: Asociada al registro `Product2` que es el producto hijo que se pretende agregar a la oferta.
* <code>start_date</code>: Fecha que se pretende dejar como fecha de inicio de las oferta de prueba.
* <code>end_date</code>: Fecha que se pretende dejar como fecha de término de las oferta de prueba.

Todas esta variables se actualizan automaticamente de acuerdo a los resultados de las solicitudes almacenadas en la colección.

### [POST] Get Auth Token

Este es el llamado que hay que realizar inicialmente para obtener el token de acceso a Salesforce. Si la configuración del entorno explicada previamente se hizo correctamente, y el usuario y la aplicación conectada tienen los permisos y configuraciones necesarias, sólo hará falta dar clic al botón *"Send"* y se obtendrá el token de acceso.

El llamado es de tipo *POST*, y su cuerpo está construido como se muestra a continuación:

| Key | Value |
|----------|:-------------:|
| grant_type | `"password"` |
| client_id |    `{{sf_client_id}}`   |
| client_secret | `{{sf_client_secret}}` |
| username | `{{sf_user_name}}` |
| password | `{{sf_password}}{{sf_user_token}}` |

donde los valores entre corchetes son determinados dinamicamente de acuerdo a lo establecido en el entorno, como ya se explicó previamente.

Si el llamado se ejecuta correctamente, la respuesta deberá ser similar a lo siguiente:

    {
        "access_token": "<<BIG HASH - YOUR TOKEN ID>>",
        "instance_url": "https://<<YOUR_INSTANCE>>.my.salesforce.com",
        "id": "https://login.salesforce.com/id/<<SOME_ID>>",
        "token_type": "Bearer",
        "issued_at": "<<1721588207274>>",
        "signature": "<<ANOTHER HASH>>="
    }

Como parte de la configuración del llamado, existe un `script` que se ejecuta después de llamado que es el siguiente:

    const jsonData = pm.response.json();
    const id = jsonData.id.split('/');

    if (!jsonData.error) {
        const context = pm.environment.name ? pm.environment : pm.collectionVariables;
        context.set("access_token", jsonData.access_token);
    }

Que basicamente establece la variable de entorno `access_token` con el valor de la variable `access_token` encontrada en el cuerpo de la respuesta.

### [GET] Get Test Account From Query

Este llamado busca a traves de la API de Salesforce la cuenta asociada al solicitante de prueba. La consulta SOQL que se realiza es la siguiente:

```sql
SELECT Id FROM Account WHERE CodigoSolicitante__c = '0000645248' LIMIT 1
```

Esta consulta se puede modificar de acuerdo a la necesidad.

De esta consulta se obtiene el Id de la cuenta asociada al solicitante, el cual se almacena en la variable de colección <code>current_account_id</code>.

### [GET] Get Test Buyer Group Id

Obtenido el Id de la cuenta asociada al solicitante, se busca el Id del grupo de compradores al que pertenece la cuenta. La consulta SOQL que se realiza es la siguiente:

```sql
SELECT BuyerGroupId FROM BuyerGroupMember WHERE BuyerId = '{{current_account_id}}'
```

De esta consulta se obtiene el Id del grupo de compradores al que pertenece la cuenta asociada al solicitante, el cual se almacena en la variable de colección <code>current_buyer_group_id</code>.

### [GET] Get Test Policy Id

Con el Id del grupo de compradores, se busca el Id de la política de comercio asociada al grupo de compradores. La consulta SOQL que se realiza es la siguiente:

```sql
SELECT PolicyId FROM CommerceEntitlementBuyerGroup WHERE BuyerGroupId = '{{current_buyer_group_id}}'
```

De esta consulta se obtiene el Id de las políticas al grupo de compradores. Aquí es necesario tener presente que esta consulta en principio retorna una lista de registros, no un único registro de política, por lo que actualmente se está tomando solamente la primera politica en la lista, y es este Id de esta política el que se almacena en la variable de colección <code>current_policy_id</code>.

### [GET] Get Test Parent Products

Esta consulta recupera los productos asociados al grupo de compradores ligados mediante la política de comercio. que esten activos y no archivados. La consulta SOQL que se realiza es la siguiente:

```sql
SELECT ProductId FROM CommerceEntitlementProduct WHERE PolicyId= '{{current_policy_id}}' AND Product.ProductClass = 'VariationParent' AND Product.IsActive = true AND Product.IsArchived = false
```

Al igual que ocurre con las políticas, esta consulta retorna una lista de productos, y se almacena el Id del primer producto en la lista en la variable de colección <code>current_parent_product_id</code>.


### [GET] Get Test Product Attributes

Dado que, de acuerdo al modelo de datos, los productos hijos estan asociados a producto padre mediante registros en el objeto `ProductAttribute`, se realiza una consulta SOQL para obtener los atributos asociados al producto padre. La consulta SOQL que se realiza es la siguiente:

```sql
SELECT ProductId FROM ProductAttribute WHERE VariantParentId = '{{current_parent_product_id}}' AND Product.IsActive = true
```
Una vez más, este consulta retorna una lista de productos, y se almacena el Id del primer producto en la lista en la variable de colección <code>current_child_product_id</code>.

### [POST] Create Deal For Parent Product

Se entiende que para que el sistema de ofertas funcione correctamente, es necesario que exista un registro de oferta asociado a un producto padre. Para esto se realiza una solicitud de tipo *POST* con el siguiente cuerpo:

```json
{
    "Account__c": "{{current_account_id}}",
    "Product__c": "{{current_parent_product_id}}",
    "EndDate__c": "{{end_date}}",
    "StartDate__c": "{{start_date}}"
}
```

Donde se asocia la cuenta, el producto padre, y las fechas de inicio y término de la oferta. Esta solicitud intentará crear un registro en el objeto `Deal__c` en Salesforce. Si la solicitud se realiza correctamente, se obtendrá un mensaje de respuesta similar a lo siguiente:

```json
{
    "id": "a2FVZ00000056ZR2AY",
    "success": true,
    "errors": []
}
```

### [POST] Create Deal For Child Product

Este llamado funciona exactamente igual que el anterior, sólamente se varía la obtención de id del producto padre por el id del producto hijo. El cuerpo de la solicitud es el siguiente:

```json
{
	"Account__c": "{{current_account_id}}",
	"Product__c": "{{current_child_product_id}}",
	"EndDate__c": "{{end_date}}",
	"StartDate__c": "{{start_date}}"
}
```