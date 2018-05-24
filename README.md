
# Tile38 Node driver

This library can be used to access the Tile38 server from Node.js apps. 


# Links
* [Project git repo](https://github.com/phulst/node-tile38)
* [Tile38 website](http://tile38.com/)
* [Tile38 Github](https://github.com/tidwall/tile38)

# Installation

```
npm install tile38
```

# Overview

While you can use your preferred Redis library to communicate with the Tile38 geolocation database, this node library 
offers a much more pleasant query interface for all Tile38 commands. It also supports live geofencing using 
persistent sockets to the server. 
 
In most cases, commands follow the [command documentation](http://tile38.com/commands/) on the Tile38 website, 
though the search/scan commands use method chaining. You can find some examples below as well as in the 
examples folder. 

Most examples below are in ES6, and the library has been written in ES6, but it uses Babel to transpile, so
even if you're still running Node 4 or earlier you should be able to use this.

This library has not been tested in the browser. You generally would not want to expose your database directly to the 
internet, so even if it does work from the browser, it's not a good idea. 
 

## Revision history

See the [CHANGELOG](./CHANGELOG.md) 

## Connection 
 
```
var Tile38 = require('tile38'); 
var client = new Tile38();
// save a location
client.set('fleet', 'truck1', [33.5123, -112.2693]);
// save a location with additional fields
client.set('fleet', 'truck2', [33.5123, -112.2693], { value: 10, othervalue: 20});
```

You can pass any non-default connection settings into the Tile38 constructor, and you can also turn on 
optional debug logging as illustrated below. 

```
var client = new Tile38({host: 'host.server.com', port: 9850, debug: true });
```

You can also set the hostname, port and password using the environment vars TILE38_HOST, TILE38_PORT and TILE38_PASSWD.
These environment variables will only be used if values are not passed into the constructor explicitly


## Promises

All of the implemented methods return promises (with the exception of executeFence(), see more info on that below). 

```
client.get('fleet', 'truck1').then(data => {
  console.log(data); // prints coordinates in geoJSON format 

}).catch(err =>
  console.log(err); // id not found  
});

// return the data as type POINT, and include FIELDS as well.  
client.get('fleet', 'truck2', {type: 'POINT', withfields: true}).then(data => {
  console.log(`truck2 is at ${data.point.lat},${data.point.lon}`);
  console.dir(data.fields);
});
// There's also a getPoint(id,key) method that can be used as a shortcut instead of get(id,key,{type:'POINT'})
// as well as similar getBounds and getHash methods. 
```

Many commands may not return values, but they still return promises, allowing you to wait until 
your changes have been persisted. 

```
client.set('fleet', 'truck1', [33.5123, -112.2693]).then(() => {
  console.log('your changes have been persisted');
});

```

# Command examples

The command documentation for Tile38 server is followed as closely as possible. Command names become function names, 
mandatory properties become arguments, and optional properties become either optional arguments or are passed in 
through an options object argument. 

For example, the command 

```
JGET key id path
```

is called as follows: 

```
client.jget(key, id, path)
```

## keys commands

Some examples of keys commands: 

```
client.bounds('fleet');
client.del('feet', 'truck2');
client.drop('fleet');
client.expire('fleet','truck', 10);
client.fset('fleet', 'truck1', 'speed', 16);
client.stats('fleet1', 'fleet2');
...etc
```

### get command
The get command accepts an optional object that can be use to set the response data type: 


```
// return truck1 location as a geoJSON object
client.get('fleet', 'truck1');
client.get('fleet', 'truck1', { type: 'OBJECT' });   // equivalent
// return as POINT (2 element array with lat/lon coordinates)
client.get('fleet', 'truck1', { type: 'POINT' });
client.getPoint('fleet', 'truck1');   // equivalent of above
// return bounding rectangle
client.get('fleet', 'truck1', { type: 'BOUNDS' });
client.getBounds('fleet', 'truck1');   // equivalent of above
// return a geohash with precision 6 (must be between 1 and 22)
client.get('fleet', 'truck1', { type: 'HASH 6' });
client.getHash('fleet', 'truck1');   // equivalent of above
client.getHash('fleet', 'truck1', { precision: 8});   // if you need different precision from default (6)

// if you want the 'get' function to return fields as well, use the 'withfields' property
client.get('fleet', 'truck1', { withfields: true }); 
```

### set command

The set command has various forms. 

set(key, id, locationObject, fields, options)

```
// set a simple lat/lng coordinate
client.set('fleet', 'truck1', [33.5123, -112.2693])
// set with additional fields
client.set('fleet', 'truck1', [33.5123, -112.2693], { field1: 10, field2: 20});
// set lat/lon/alt coordinates, and expire in 120 secs
client.set('fleet', 'truck1', [33.5123, -112.2693, 120.0], null, {expire: 120})
// set bounds
set('props', 'house1', [33.7840, -112.1520, 33.7848, -112.1512])
// set an ID by geohash
set('props', 'area1', '9tbnwg')   // assumes HASH by default if only one extra parameter
// set a String value
set('props', 'area2', 'my string value', null, {type: 'string'}) # or force to String type
// set with geoJson object
set('cities', 'tempe', geoJsonObject)
// only set truck1 if it doesn't exist yet
client.set('fleet', 'truck1', [33.5123, -112.2693], null, {onlyIfNotExists: true})
```

### search commands

The search commands use method chaining to deal with its many available options. See the query examples below, or look at 
tile38_query.js to see all available methods.
  

#### One time results vs live geofence 

To execute the query and get the search results, use the execute() function, which will return a promise to the results. 
  
```  
let query = client.intersectsQuery('fleet').bounds(33.462, -112.268, 33.491, -112.245);
query.execute().then(results => {
    console.dir(results);  // results is an object.
}).catch(err => {
    console.error("something went wrong! " + err);
)};
```

To set up a live geofence that will use a websocket to continuously send updates, you construct your query the exact same way. 
However, instead of execute() (which returns a Promise), use the executeFence() function while passing in a callback function. 
 
```  
let query = client.intersectsQuery('fleet').detect('enter','exit').bounds(33.462, -112.268, 33.491, -112.245);
let fence = query.executeFence((err, results) => {
    // this callback will be called multiple times
    if (err) {
        console.error("something went wrong! " + err);
    } else {
        console.dir(results);
    }
});

// if you want to be notified when the connection gets closed, register a callback function with onClose()
fence.onClose(() => {
    console.log("geofence was closed");
});

// later on, when you want to close the socket and kill the live geofence: 
fence.close();
```

Many of the chaining functions below can be used for all search commands. See the [Tile38 documentation](http://tile38.com/commands/#search)
for more info on what query criteria are supported by what commands.

Do not forget to call the execute() or executeFence() function after constructing your query chain. I've left this out in the
examples below for brevity. 

#### INTERSECTS

```
// basic query that uses bounds
client.intersectsQuery('fleet').bounds(33.462, -112.268, 33.491 -112.245)
// using cursor and limit for pagination
client.intersectsQuery('fleet').cursor(100).limit(50).bounds(33.462, -112.268, 33.491 -112.245)
// create a fence that triggeres when entering a polygon
let polygon = {"type":"Polygon","coordinates": [[[-111.9787,33.4411],[-111.8902,33.4377],[-111.8950,33.2892],[-111.9739,33.2932],[-111.9787,33.4411]]]};
client.intersectsQuery('fleet').detect('enter','exit').object(polygon)
```

#### SEARCH

``` 
// basic search query
client.searchQuery('names')
// use matching patter and return results in descending order, without fields
client.searchQuery('names').match('J*').nofields().desc()
// return only IDs
client.searchQuery('names').output('ids')
// this does the same: 
client.searchQuery('names').ids()
// return only count
client.searchQuery('names').count()
// use the where option
client.searchQuery('names').where('age', 40, '+inf')
```

#### NEARBY

```
// basic nearby query, including distance for each returned object
client.nearbyQuery('fleet').distance().point(33.462, -112.268, 6000)
// return results as geohashes with precision 8
client.nearbyQuery('fleet').point(33.462, -112.268, 6000).hashes(8)
// use the roam option
client.nearbyQuery('fleet').roam('truck', 'ptn', 3000)
```

#### SCAN

```
// basic scan query, returning all results in geojson
client.scanQuery('fleet').output('objects');
// this does the same
client.scanQuery('fleet').objects()
// return simple coordinates, and do not include fields
client.scanQuery('fleet').nofields().points()
```

#### WITHIN

The withinQuery has the same query options as intersects.

```
// basic within query, returning all results in geojson
client.withinQuery('fleet').bounds(33.462, -112.268, 33.491, -112.245)
// check within an area that's already defined in the database
client.withinQuery('fleet').get('cities', 'tempe')
// return objects within a given tile x, y, z
client.withinQuery('fleet').tile(x, y, z); 
```


# Running tests

WARNING: THIS WILL WIPE OUT YOUR DATA!
The test suite currently depends on having a local instance of Tile38 running on port 9850 (instead of the default 9851).
It tests all supported commands, including FLUSHDB, so you'll LOSE ALL EXISTING DATA in your local db.

(I changed the default port for the test suite to make it less likely that someone accidentally runs the test suite
on a local database containing critical data.) 
 
If you have nothing critical in your local db, you can run the tests with: 

```
npm test
```

# Project roadmap

The following work or features are up next, in order of priority, high to low:  

- The executeFence method / live geofences has had limited testing. Please submit bugs if you run into issues. 
- The SETHOOK command has some similarities to the search commands. It's not currently using method chaining but I may rewrite 
  it so it can be used in a similar way to the other search functions. 
- Replication Commands (AOF, AOFMD5, AOFSHRINK, FOLLOW)
- There's virtually no validation of options passed into functions, or of the query chaining for search commands. For example,
  this library does not prevent you from using the roam() function on an intersects query, even though it's only supported 
  on the nearby search. It may be fine to leave validation of the search query up to the Tile38 server itself. Happy to accept
  a pull request for better query validation though. 

Testing TODO:
- webhooks needs test coverage
- test coverage for live geofences

# Missing something? Did it break?

For bugs or feature requests, please open an issue.  
