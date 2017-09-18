---
layout: post
title: "Using the Mongoose framework with Cosmos DB"
comments: true
date: 2017-09-10
---

Azure Cosmos DB is Microsoft’s globally distributed multi-model database service. You can quickly create and query document, key/value, and graph databases, all of which benefit from the global distribution and horizontal scale capabilities at the core of Azure Cosmos DB.

This quickstart demonstrates how to use the [Mongoose Framework](http://mongoosejs.com/) when storing data in Cosmos DB. We will use the MongoDB API for Cosmos DB for this walkthrough. For those of you unfamiliar, Mongoose is an onject modeling framework for MongoDB in NodeJS and provides a straight-forward, schema-based solution to model your application data.

## Prerequisites

* Before you can run this sample, you must have the following prerequisites:
    * NodeJS
* If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.
* Alternatively, you can create a [Try Azure Cosmos DB](https://azure.microsoft.com/try/cosmosdb/) subscription, or you can use the [Azure Cosmos DB Emulator](https://docs.microsoft.com/en-us/azure/cosmos-db/local-emulator) for this 
tutorial with a username of localhost:10255 and a key of   ```C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==```
* A Cosmos DB Account (MongoDB API) & collection already created. You can [follow the instructions here to create one](https://docs.microsoft.com/en-us/azure/cosmos-db/create-mongodb-java#create-a-database-account).

## Code

If you'd like to skip the walkthrough and take a look at the sample (or clone it). The repo can be found here: [Mongoose Cosmos DB Github Sample](https://github.com/romitgirdhar/AzureSamples/tree/master/Mongoose_CosmosDB)

## Setup

1. Create a NodeJS application in the folder of your choice
    ``` npm init ```
    Answer the questions and your project will be ready to go.

1. Add a new file to the folder called ```index.js```.
1. Install the necessary packages using ```npm install```
    1. Mongoose: ```npm install mongoose --save ```
    1. Dotenv (if you'd like to load your secrets from a .env file): ```npm install dotenv --save ```
    >**Note**: the ```--save``` flag will add the dependency to the package.json file.

1. Import the dependencies in your index.js
    ``` 
    var mongoose = require('mongoose');
    var env = require('dotenv').load();
    ```

1. Connect to Cosmos DB using the Mongoose
    ``` 
    mongoose.connect(process.env.COSMOSDB_CONNSTR+process.env.COSMOSDB_DBNAME+"?ssl=true&replicaSet=globaldb"); //Creates a new DB, if it doesn't already exist

    var db = mongoose.connection;
    db.on('error', console.error.bind(console, 'connection error:'));
    db.once('open', function() {
    // we're connected!
    console.log("Connected to DB");
    });
    ```
    >**Note**: Here, the environment variables are loaded using the process.env.{variableName}

    Once we're connected to our Cosmos DB, we can now start setting up Object models in Mongoose. 

## Caveats with using Mongoose with Cosmos DB

For every model you create, Mongoose creates a new MongoDB collection underneath the covers. However, given the per-collection billing model of Cosmos DB, it might not be the most cost-efficient way to go, if you've got multiple object models that are sparsely populated.

In this walkthrough, we will cover both models. We'll first cover the walkthrough on storing one type of data per collection. This is the defacto behavior for Mongoose.

Mongoose also has the notion of [Discriminators](http://mongoosejs.com/docs/discriminators.html). Discriminators are a schema inheritance mechanism. They enable you to have multiple models with overlapping schemas on top of the same underlying MongoDB collection.

You can store the various data models in the same collection and then use a filter clause at query time to pull down only the data needed.

### One Collection Per Object Model

The default Mongoose behavior is to create a MongoDB collection every time you create an Object model. In this section, we will explore how to achieve this with MongoDB for Cosmos DB. This method is recommended with Cosmos DB when you have object models with large amounts of data (Roughly ~ 5GB data, at least). This is the default operating model for Mongoose, so, you might be familar with this, if you're familiar with Mongoose.

1. Open your ```index.js``` again.

1. Create the schema definition for 'Family'.

```JavaScript
const Family = mongoose.model('Family', new mongoose.Schema({
    lastName: String,
    parents: [{
        familyName: String,
        firstName: String,
        gender: String
    }],
    children: [{
        familyName: String,
        firstName: String,
        gender: String,
        grade: Number
    }],
    pets:[{
        givenName: String
    }],
    address: {
        country: String,
        state: String,
        city: String
    }
}));
```

1. Create an object for 'Family'.

```JavaScript
const family = new Family({
    lastName: "Andersen",
    parents: [
            { firstName: "Thomas" },
            { firstName: "Mary Kay" }
        ],
        children: [
            { firstName: "John", gender: "male", grade: 7 }
        ],
        pets: [
            { givenName: "Fluffy" }
        ],
        address: { country: "USA", state: "WA", city: "Seattle" }
});
```

1. Finally, let's save the object to Cosmos DB. This will create a collection underneath the covers.

```JavaScript
family.save((err, saveFamily) => {
    console.log(JSON.stringify(saveFamily));
});
```

1. Now, let's create another schema and object. This time, let's create one for 'Vacation Destinations' that the families might be interested in.
    1. Just like last time, let's create the scheme
    ```JavaScript
    const VacationDestinations = mongoose.model('VacationDestinations', new mongoose.Schema({
        name: String,
        country: String
    }));
    ```

    1. Create a sample object (You can add multiple objects to this schema) and save it.
    ```JavaScript
    const vacaySpot = new VacationDestinations({
        name: "Honolulu",
        country: "USA"
    });

    vacaySpot.save((err, saveVacay) => {
        console.log(JSON.stringify(saveVacay));
    });
    ```

1. Now, going into the Azure portal, we can see that we've got 2 collections created in Cosmos DB.

//TODO: Add Screenshot of collections.

1. Finally, let's read the data from Cosmos DB. Since we're using the default Mongoose operating model, the reads will be the same with any other reads with Mongoose.
    ```JavaScript
    Family.find({ 'children.gender' : "male"}, function(err, foundFamily){
        foundFamily.forEach(fam => console.log("Found Family: " + JSON.stringify(fam)));
    });
    ```

### Using Mongoose Discrimintors to Store data in a single collection

In this method, we will use [Mongoose Discriminators](http://mongoosejs.com/docs/discriminators.html) to help optimize for the costs of each Cosmos DB collection. Discriminators allow us to define a differentiating 'Key', which will allow us to store, differentiate and filter on different object models.

Here, we will create a base object model, define a differentiating key and add our 'Family' and 'VacationDestinations' as an extension to our base model.

1. Let's setup our base config and define our discriminator key.

```JavaScript
const baseConfig = {
    discriminatorKey: "_type", //If you've got a lot of different data types, you could also consider setting up a secondary index here.
    collection: "alldata"   //Name of the Common Collection
};
```

1. Next, let's define the common object model

```JavaScript
const commonModel = mongoose.model('Common', new mongoose.Schema({}, baseConfig));
```

1. We will now define our 'Family' model. Notice here that we're using ```commonModel.discriminator``` instead of ```mongoose.model```. Additionally, we're also adding the base config to our mongoose schema. So, here, our discriminatorKey will be ```FamilyType```.

```
const Family_common = commonModel.discriminator('FamilyType', new mongoose.Schema({
    lastName: String,
    parents: [{
        familyName: String,
        firstName: String,
        gender: String
    }],
    children: [{
        familyName: String,
        firstName: String,
        gender: String,
        grade: Number
    }],
    pets:[{
        givenName: String
    }],
    address: {
        country: String,
        state: String,
        city: String
    }
}, baseConfig));
```

1. Similarly, let's add another schema, this time for our 'VacationDestinations'. Here, our DiscriminatorKey is ```VacationDestinationsType```.

```JavaScript
const Vacation_common = commonModel.discriminator('VacationDestinationsType', new mongoose.Schema({
    name: String,
    country: String
}, baseConfig));
```

1. Finally, let's create objects for the model and save it.
    1. Let's add object(s) to our 'Family' model.
    ```JavaScript
    const family_common = new Family_common({
        lastName: "Andersen",
        parents: [
            { firstName: "Thomas" },
            { firstName: "Mary Kay" }
        ],
        children: [
            { firstName: "John", gender: "male", grade: 7 }
        ],
        pets: [
            { givenName: "Fluffy" }
        ],
        address: { country: "USA", state: "WA", city: "Seattle" }
    });

    family_common.save((err, saveFamily) => {
        console.log("Saved: " + JSON.stringify(saveFamily));
    });
    ```

    1. Next, let's add object(s) to our 'VacationDestinations' model and save it.
    ```JavaScript
    const vacay_common = new Vacation_common({
        name: "Honolulu",
        country: "USA"
    });

    vacay_common.save((err, saveVacay) => {
        console.log("Saved: " + JSON.stringify(saveVacay));
    });
    ```

1. Now, if we go to the Azure portal, we'll notice that we have only 1 collection called ```alldata``` with both 'Family' and 'VacationDestinations' data.

//TODO: Add Screenshot of collections.

1. Also, notice that each object has another attribute called as ```__type```, which help us differentiate between our different object models.

1. Finally, let's read the data we just stored in Cosmos DB. Mongoose takes care of filtering data based on the model. So, you have to do nothing different when reading data. Just specify your model (in this case, ```Family_common```) and Mongoose will handle filtering on the 'DiscriminatorKey'.

```JavaScript
Family_common.find({ 'children.gender' : "male"}, function(err, foundFamily){
    foundFamily.forEach(fam => console.log("Found Family (using discriminator): " + JSON.stringify(fam)));
});
```

As you can see, it is very easy to work with Mongoose discriminators. So, if you have an app that is already using the Mongoose framework, this is a way for you to get your application up and running on Azure Cosmos DB without having to make too many changes.

## Clean up resources

If you're not going to continue to use this app, use the following steps to delete all resources created by this tutorial in the Azure portal. 
1. From the left-hand menu in the Azure portal, click **Resource groups** and then click the name of the resource you created. 
1. On your resource group page, click **Delete**, type the name of the resource to delete in the text box, and then click **Delete**.

## Conclusion

As mentioned earlier, the code for this tutorial can be found here: [Mongoose Cosmos DB Github Sample](https://github.com/romitgirdhar/AzureSamples/tree/master/Mongoose_CosmosDB)

In summary, we learnt how to work with MongoDB API on Cosmos DB using the Mongoose framework. We looked at two different approaches, viz. the default approach and the discriminator approach.