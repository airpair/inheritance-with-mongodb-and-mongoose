I use [Mongoose](http://mongoosejs.com/) - the ORM for MongoDB, everyday, about it and there's one thing in Mongoose, I find very extremely useful - **discriminators**. In a nutshell, it's inheritance of database models but the documentation for it has a lot to be desired.  

I was working on a model that made use of discriminators and like before, I thought it would be easy sailing but I found that populating child fields from the parent/base schema was not possible with the version I was using and after a lot of *Googling* and after sifting through the tons of issues on *GitHub*, I found that it was introduced in the latest release. This is my try at explaining things better.  

###So what's the deal  
In classical OOP principles, lets say we have a base model for *Vehicles*. So, for every Vehicle, we have fields that will be common for every other model that will inherit from it. Some features that all Vehicles will have -  
  
- Type  
- Model  
- Year  
- Mileage  
- BHP  
- Engine Type    

Then, we can start declaring models for each individual type of Vehicle, e.g Toyota, Honda and so on. We can add individual fields for each of these vehicle types and the world is suddenly a better place!  

###Where's the code already?!  
Enough talking! More coding!  

**models/Vehicle.js**  
```  
var util = require('util');  
var mongoose = require('mongoose');  
var Schema = mongoose.Schema;  

function BaseSchema() {  
    Schema.apply(this, arguments);  

    this.add({  
        model: { type: Schema.Types.ObjectId, ref: 'Model', required: true },  
        year: { type: Number },  
        mileage: { type: String },  
        bhp: { type: Number },  
        engine: { type: String }  
    });  
}  

util.inherits(BaseSchema, Schema);  
```  

First thing first, `util`, is the built-in library in Node with lots of useful methods but for this, we are looking at `util.inherits` which lets you *inherit prototype methods from one constructor to another*. So for every schema that we define using our `BaseSchema`, it will inherit all fields, methods and statics, which I've done in the last line. The `BaseSchema` is just the default fields that all other models will include but notice that the `model` field is an `ObjectId` reference because we want to be able to populate the models for a car type (the Toyota Corolla has a thousand different faces), when we need to.  

But where's the type? Mongoose has a nifty feature called *Virtuals* and by default, all discriminator models have a virtual property `__t` which is actually the reference to the type of the schema. So, in our model file above:  
```  
var VehicleSchema = new BaseSchema();

VehicleSchema.virtual('type').get(function () { return this.__t; });  
```  

Now, every time you do `doc.type`, it will return the type of Schema. Moving on,  
```  
var ToyotaSchema = new BaseSchema({ origin: {type: String} });  
var AudiSchema = new BaseSchema({ doors: {type: Schema.Types.ObjectId} });  

// finishing up the file  
mongoose.model('Vehicle', VehicleSchema);
Vehicle.discriminator('Toyota', ToyotaSchema);  
Vehicle.discriminator('Audi', AudiSchema);   
```  

We, define each individual car type now with each having their own unique feature but that each car type is inherited from the `Vehicle` schema using the `Schema.discriminator` method. So, for Toyota, we have an origin field and for Audi, a list of `Ids` which refer to car door types, and finish up with initialising the models with Mongoose.  

**controllers/vehicles.js**  
```  
var mongoose = require('mongoose');  
var Vehicle = mongoose.model('Vehicle');  
var Toyota = mongoose.model('Toyota');  
var Audi = mongoose.model('Audi');  
```  

In our controller, we `require` each model separately the usual way but how we query these models is where it gets interesting.  
```  
Vehicle.find({}).exec(callback);  
// {  
      type: Toyota,  
      origin: Japan,  
      ...  
   },  
   {  
     type: Audi,  
     ...  
   }  
```  

Running a find query on the `Vehicle` model returns all the different types of vehicles we described. If we wanted to query a single vehicle type, we could either pass in a type parameter when querying the `Vehicle` schema or query each `Vehicle` model separately - `Toyota.find({}).exec(callback);`.  

###The nitty gritty  
All that I have discussed so far is the basic. Now, lets talk about what I was talking about in the very beginning - about populating child fields from the parent model. Remember, how we had a different model for door types and then used them in the `Audi` model? If we were querying the parent schema for Audis and we needed the populated data for door types, it would just be as straight forward as populating field references for normal models starting from the latest releases of 3.8 and 4.0.2+.  
```  
Vehicle.find({ type: 'Audi' }).populate('doors').exec(callback);  
// {  
      type: Audi,  
      doors: ['Scissors', 'Suicide', 'Sliding'],  
      ...  
   }  
```  

And there you have it. You can now populate data for references from anywhere whether you are searching from the parent or the individual child model.


