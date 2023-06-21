# Descarga del enunciado

Puedes descargar el enunciado en formato PDF desde [este enlace](enlace al PDF).

## Introducción

Aquí tienes el enunciado del módulo 2. Crea un repo en Github y añade un `readme.md` incluyendo enunciado y consulta (lo que pone aquí Pega aquí tu consulta).

## Basico

### Restaurar backup

Vamos a restaurar el set de datos de mongo atlas airbnb.

Lo puedes encontrar en este enlace: [enlace al backup](https://drive.google.com/drive/folders/1gAtZZdrBKiKioJSZwnShXskaKk6H_gCJ?usp=sharing).

Para restaurarlo, puedes seguir las instrucciones de este [videopost](enlace al videopost).

Acuérdate de mirar si en `opt/app` hay contenido de backups previos que tengas que borrar.

## General

En esta base de datos puedes encontrar un montón de apartamentos y sus reviews, esto está sacado de hacer webscrapping.

Pregunta: Si montaras un sitio real, ¿qué posibles problemas potenciales le ves a cómo está almacenada la información?

Hay datos como por ejemplo, las ciudades, que estan en varios campos y mal ubicados. Algunas ciudades no concuerdan con su país de origen.

## Consultas

### Basico

NOTA: Antes de todas las queries hay que lanzar el comando: 
```
use("airbnb");
```

- Saca en una consulta cuántos apartamentos hay en España.

```
db.listingsAndReviews.find({ "address.country": "Spain" }).count();
```



- Lista los 10 primeros:
Sólo muestra: nombre, camas, precio, government_area
Ordenados por precio.

```
db.listingsAndReviews
    .find({},{ _id:0,name: 1, beds: 1, price: 1, "address.government_area": 1 })
    .sort({ price: 1 })
    .limit(10);
```

### Filtrando

Queremos viajar cómodos, somos 4 personas y queremos:
- 4 camas.
- Dos cuartos de baño.

```
db.listingsAndReviews.find({
    beds: 4,
    bathrooms: 2
  })
```  

Al requisito anterior, hay que añadir que nos gusta la tecnología y queremos que el apartamento tenga wifi.

```
db.listingsAndReviews.find({
    beds: 4,
    bathrooms: 2,
    amenities: { $in: ["Wifi"]}
  })
```

Y bueno, un amigo se ha unido que trae un perro, así que a la query anterior tenemos que buscar que permitan mascotas (Pets Allowed).

```
db.listingsAndReviews.find({
    beds: 4,
    bathrooms: 2,
    amenities: { $all: ["Wifi","Pets allowed"]}
  })
  ```

### Operadores lógicos

Estamos entre ir a Barcelona o a Portugal, los dos destinos nos valen, peeero... queremos que el precio nos salga baratito ($50), y que tenga buen rating de reviews.
```
db.listingsAndReviews.find({
  $or: [
    { "host.host_location": { $regex: "Barcelona" } },
    { "address.country": "Portugal" },
  ],
  price: { $lte: 50 },
  "review_scores.review_scores_rating": { $gt: 90 },
});
```
### Agregaciones

Queremos mostrar los pisos que hay en España, y los siguientes campos:
- Nombre.
- De qué ciudad (no queremos mostrar un objeto, solo el string con la ciudad).
- El precio.
```
db.listingsAndReviews.aggregate(
{
   $match:{"address.country": "Spain"} 
},
{
  $project: { _id: 0, 
    name: 1,
    location:"$host.host_location",
    price: { $toDouble: "$price" }
    }
});
```

Queremos saber cuántos alojamientos hay disponibles por país.
```
db.listingsAndReviews.aggregate([
  {
    $match: {
      $or: [
        { "availability.availability_30": { $gt: 0 } },
        { "availability.availability_60": { $gt: 0 } },
        { "availability.availability_90": { $gt: 0 } },
        { "availability.availability_365": { $gt: 0 } },
      ],
    },
  },
  {
    $group: {
      _id: "$address.country",
      count: { $sum: 1 },
    },
  },
]);
  ```

### Opcional

Queremos saber el precio medio de alquiler de Airbnb en España.
```
use("airbnb");


db.listingsAndReviews.aggregate([
  {
    $match: {
      "address.country": "Spain"
    }
  },
  {
    $group: {
      _id: null,
      mediaEspaña: { $avg: { $toDouble: "$price" } },
      count: { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      mediaEspaña: 1,
      count: 1
    }
  }
]);
```

¿Y si quisiéramos hacer como el anterior, pero sacarlo por países?
```
db.listingsAndReviews.aggregate([

  {
    $group: {
      _id: "$address.country",
      mediaPrecio: { $avg: { $toDouble: "$price" } },
    }
  },
  {
    $project: {
      _id: 0,
      pais: "$_id",
      mediaPrecio:1
    }
  }
]);
```


Repite los mismos pasos pero agrupando también por número de habitaciones.
use("airbnb");

```
db.listingsAndReviews.aggregate([

    {
      $group: {
        _id:{pais: "$address.country",numHabitaciones:"$bedrooms"},

        mediaPrecio: { $avg: { $toDouble: "$price" } },

       
      }
    },
    {
      $project: {
        _id: 0,
        pais: "$_id.pais",
        numHabitaciones: "$_id.numHabitaciones",
        mediaPrecio:1
      }
    },
     {
        $sort:{
            pais:1,
            numHabitaciones:1
        }
     }
]);
```    


### Desafio

Queremos mostrar el top 5 de apartamentos más caros en España, y sacar los siguientes campos:
- Nombre.
- Ciudad.
- Amenities, pero en vez de un array, un string con todos los amenities.

```
db.listingsAndReviews.aggregate([
    {
      $match: {
        "address.country": "Spain"

      }
    },
    
    {
        $project: {
          _id: 0,
          ciudad:"$host.host_location",
          price: { $toDouble: "$price" },
          extras:{
            $reduce:{
                input:"$amenities",
                initialValue:"",
                in:{
                    $concat:[
                        "$$value",
                    { $cond: [{ $eq: ["$$value", ""] }, "", ", "] },
                        "$$this"
                    ]

                }}
            }}
        },
        {
            $sort:{
                price:-1,
            }
         },
         {
            $limit:5,
         }
  ]);
```
