# Environment State Model

The purpose of this document is to outline the feature requirements and implementation details of an environment state model to be used to track changes in the environment such as deploying services, modifying service settings, scaling services, etc.

## Requirements

- Instantiable
  - Needs to be able to be individually instantiated to allow for easy testing
- Receive environment delta commands.
- Save hierarchical deltas.
  - Changing service settings must come after a service is deployed.
- Remove hierarchical deltas.
  - If you remove a newly added service it should remove all changes to that service.
- Remove individual deltas.
- Clear-all command.


## Implementation

Every environment will have a single instantiation of the ESM. Changes to that environment will interact directly with the ESM instead of calling to the environment, as it currently does, to add to the state change queue. When requested the ESM will be able to output an object structure which will be parsed and converted into Juju environment calls.

The ESM provides a set of convenience methods on top of a persistent data store library (currently a POJI) who's responsibility it is to store and modify the deltas sent by the ESM.

### API
All methods have two parameters, config and options. The structure of the config parameter matches the appropriate method in the environment. The options paramater are for options related to the ESM instance such as the 'immediate' boolean property which executes the command immediately without adding it to the queue.

#### Constructor
`EnvironmentStateModel(config Object) instance Object`

#### Command syntax
`deploy(config Object, options Object) id Int`

#### Convenience methods
`queueList() list Array`

`queueListOverview() list Array`

`remove(tmpId String) record Object`

#### Queue Example
```javascript
var esm = new EnvironmentStateModel();

var tmpServiceId = esm.deploy({
  charmUrl: '',
  serviceName: '',
  config: {},
  configRaw: '',
  numUnits: 0,
  constraints: {},
  toMachine: '',
  callback: function() {}
});

esm.setConfig({
  // If you want to setConfig on a deployed service you would pass a real id
  // instead of a temporary service already in the queue.
  serviceName: tmpServiceId,
  config: {},
  data: '',
  serviceCOnfig: {},
  callback: function() {}
}, {
  immediate: true
})
```

### Code
```javascript
paramStructures: {
  deploy: ['charmUrl', 'serviceName', 'config', 'configRaw', 'numUnits',
    'constraints', 'toMachine', 'callback']
},

_deconstruct: function(method, config) {
  var deconConfig = [];
  this.paramStructures[method].forEach(function(val) {
    deconConfig.push(config[val]);
  });
  return deconConfig
},

_execute: function(method, config) {
  var env = this.env;
  env[method].apply(env, this._deconstruct(method, config));
},

_generateKey: function() {
  return Math.floor((Math.random()*1000)+1);
},

_generateUniqueKey: function() {
  var key = this._generateKey();
  while (this.pds[key] !== undefined) {
    key = this._generateKey();
  }
  return key;
},

_createNewRecord: function(type) {
  var key = this._generateUniqueKey();
  this.pds[key] = {};
  return key;
},

_fetchService: function(serviceName) {
  var service = this.db.services.getById(serviceName);
  if (service) {
    return service;
  } else if (this.pds[serviceName]) {
    return this.pds[serviceName];
  } else {
    return null;
  }
}

_createService: function(config) {
  var key = this._createNewRecord('service');
  var record = this.pds[key];
  record.commands = [];
  record.commands.push({
    command: 'deploy',
    config: config
  });
  return key;
},

_setConfig: function(config) {
  var serviceName = config.serviceName;
  var service = this._fetchService(serviceName);
  if (service !== null) {
    if (service.set) {
      // If it's a deployed service
      this._execute('set_config', config);
    } else {
      // it's a queued service
      this.pds[serviceName].commands.push({
        command: 'set_config',
        config: config
      });
    }
  }
},

deploy: function(config, options) {
  // If they want to execute the command immediately.
  if (options.immediate) {
    this._execute('deploy', config);
    return; // Callback handles everything from here, no id's returned;
  }
  return this._createService(config);
}

setConfig: function(config, options) {
  if (options.immediate) {
    this._execute('set_config', config); // not implemented in prototype
    return; // Callback handles everything from here, no id's returned;
  }
  return this._setConfig(config);
}
```
