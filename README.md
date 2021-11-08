[![Go Report Card](https://goreportcard.com/badge/github.com/ohbonsai/gogm)](https://goreportcard.com/report/github.com/ohbonsai/gogm)
[![Actions Status](https://github.com/ohbonsai/gogm/workflows/Go/badge.svg)](https://github.com/ohbonsai/gogm/actions)
[![GoDoc](https://godoc.org/github.com/ohbonsai/gogm?status.svg)](https://godoc.org/github.com/ohbonsai/gogm)
# GoGM Golang Object Graph Mapper v2

```
go get -u github.com/ohbonsai/gogm/v2
```

## Features
- Struct Mapping through the `gogm` struct decorator
- Full support for ACID transactions
- Underlying connection pooling
- Support for HA Casual Clusters using `bolt+routing` through the [Official Neo4j Go Driver](https://github.com/neo4j/neo4j-go-driver)
- Custom queries in addition to built in functionality
- Builder pattern cypher queries using [MindStand's cypher dsl package](https://github.com/mindstand/go-cypherdsl)
- CLI to generate link and unlink functions for gogm structs.
- Multi database support with Neo4j v4

## What's new in V2
- GoGM is an object now! This means you can have multiple instances of GoGM at a time
- OpenTracing and Context Support
- Driver has been updated from v1.6 to v4
- Log interface, so anyone can use the logger of their choice instead of being forced to use logrus
- Primary Key strategies to use any type of primary key. GoGM is no longer UUID only!
- TLS now supported


## Usage

### Primary Key Strategy
Primary key strategies allow more customization over primary keys. A strategy is provided to gogm on initialization.
<br/>
Built in primary key strategies are:
- gogm.DefaultPrimaryKeyStrategy -- just use the graph id from neo4j as the primary key
- gogm.UUIDPrimaryKeyStrategy -- uuid's as primary keys
```go
// Example of the internal UUID strategy
PrimaryKeyStrategy{
	// StrategyName is the name to reference the strategy
    StrategyName: "UUID",
    // DBName is the name of the field in the database
    DBName:       "uuid",
    // FieldName is the name of the field in go, this will validate to make sure all pk's use the same field name
    FieldName:    "UUID",
    // Type is the reflect type of the primary key, in this case it's a string but it can be any primitive
    Type:         reflect.TypeOf(""),
    // GenIDFunc defines how new ids are generated, in this case we're using googles uuid library
    GenIDFunc: func() (id interface{}) {
        return uuid.New().String()
    },
}
```

### Struct Configuration
##### <s>text</s> notates deprecation

Decorators that can be used
- `name=<name>` -- used to set the field name that will show up in neo4j.
- `relationship=<edge_name>` -- used to set the name of the edge on that field.
- `direction=<INCOMING|OUTGOING|BOTH|NONE>` -- used to specify direction of that edge field.
- `index` -- marks field to have an index applied to it.
- `unique` -- marks field to have unique constraint.
- `pk=<strategy_name>` -- marks field as a primary key and specifies which pk strategy to use. Can only have one pk, composite pk's are not supported.
- `properties` -- marks that field is using a map. GoGM only supports properties fields of `map[string]interface{}`, `map[string]<primitive>`, `map[string][]<primitive>` and `[]<primitive>`
- `-` -- marks that field will be ignored by the ogm

#### Not on relationship member variables
All relationships must be defined as either a pointer to a struct or a slice of struct pointers `*SomeStruct` or `[]*SomeStruct`

Use `;` as delimiter between decorator tags.

Ex.

```go
type TdString string

type MyNeo4jObject struct {
  // provides required node field
  // use gogm.BaseUUIDNode if you want to use UUIDs
  gogm.BaseNode

  Field string `gogm:"name=field"`
  Props map[string]interface{} `gogm:"properties;name=props"` //note that this would show up as `props.<key>` in neo4j
  IgnoreMe bool `gogm="-"`
  UniqueTypeDef TdString `gogm:"name=unique_type_def"`
  Relation *SomeOtherStruct `gogm="relationship=SOME_STRUCT;direction=OUTGOING"`
  ManyRelation []*SomeStruct `gogm="relationship=MANY;direction=INCOMING"`
}

```

### GOGM Usage
- Edges must implement the [Edge interface](https://github.com/ohbonsai/gogm/blob/master/interface.go#L28). View the complete example [here](https://github.com/ohbonsai/gogm/blob/master/examples/example.go). 
```go
package main

import (
	"github.com/ohbonsai/gogm/v2"
	"time"
)

type tdString string
type tdInt int

//structs for the example (can also be found in decoder_test.go)
type VertexA struct {
	// provides required node fields
	gogm.BaseNode

    TestField         string                `gogm:"name=test_field"`
	TestTypeDefString tdString          `gogm:"name=test_type_def_string"`
	TestTypeDefInt    tdInt             `gogm:"name=test_type_def_int"`
	MapProperty       map[string]string `gogm:"name=map_property;properties"`
	SliceProperty     []string          `gogm:"name=slice_property;properties"`
    SingleA           *VertexB          `gogm:"direction=incoming;relationship=test_rel"`
	ManyA             []*VertexB        `gogm:"direction=incoming;relationship=testm2o"`
	MultiA            []*VertexB        `gogm:"direction=incoming;relationship=multib"`
	SingleSpecA       *EdgeC            `gogm:"direction=outgoing;relationship=special_single"`
	MultiSpecA        []*EdgeC          `gogm:"direction=outgoing;relationship=special_multi"`
}

type VertexB struct {
	// provides required node fields
	gogm.BaseNode

	TestField  string     `gogm:"name=test_field"`
	TestTime   time.Time  `gogm:"name=test_time"`
	Single     *VertexA   `gogm:"direction=outgoing;relationship=test_rel"`
	ManyB      *VertexA   `gogm:"direction=outgoing;relationship=testm2o"`
	Multi      []*VertexA `gogm:"direction=outgoing;relationship=multib"`
	SingleSpec *EdgeC     `gogm:"direction=incoming;relationship=special_single"`
	MultiSpec  []*EdgeC   `gogm:"direction=incoming;relationship=special_multi"`
}

type EdgeC struct {
	// provides required node fields
	gogm.BaseNode

	Start *VertexA
	End   *VertexB
	Test  string `gogm:"name=test"`
}

func main() {
	// define your configuration
	config := gogm.Config{
		Host:                      "0.0.0.0",
		Port:                      7687,
		// deprecated in favor of protocol
	    // IsCluster:                 false,
	    Protocol:                  "neo4j", //also supports neo4j+s, neo4j+ssc, bolt, bolt+s and bolt+ssc
	    // Specify CA Public Key when using +ssc or +s
	    CAFileLocation: "my-ca-public.crt",
		Username:                  "neo4j",
		Password:                  "password",
		PoolSize:                  50,
		IndexStrategy:             gogm.VALIDATE_INDEX, //other options are ASSERT_INDEX and IGNORE_INDEX
		TargetDbs:                 nil,
		// default logger wraps the go "log" package, implement the Logger interface from gogm to use your own logger
		Logger:             gogm.GetDefaultLogger(),
		// define the log level
		LogLevel:           "DEBUG",
		// enable neo4j go driver to log
		EnableDriverLogs:   false,
		// enable gogm to log params in cypher queries. WARNING THIS IS A SECURITY RISK! Only use this when debugging
		EnableLogParams:    false,
		// enable open tracing. Ensure contexts have spans already. GoGM does not make root spans, only child spans
		OpentracingEnabled: false,
	}

	// register all vertices and edges
	// this is so that GoGM doesn't have to do reflect processing of each edge in real time
	// use nil or gogm.DefaultPrimaryKeyStrategy if you only want graph ids
	// we are using the default key strategy since our vertices are using BaseNode
	_gogm, err := gogm.New(&config, gogm.DefaultPrimaryKeyStrategy, &VertexA{}, &VertexB{}, &EdgeC{})
	if err != nil {
		panic(err)
	}

	//param is readonly, we're going to make stuff so we're going to do read write
	sess, err := _gogm.NewSessionV2(gogm.SessionConfig{AccessMode: gogm.AccessModeWrite})
	if err != nil {
		panic(err)
	}

	//close the session
	defer sess.Close()

	aVal := &VertexA{
		TestField: "woo neo4j",
	}

	bVal := &VertexB{
		TestTime: time.Now().UTC(),
	}

	//set bi directional pointer
	bVal.Single = aVal
	aVal.SingleA = bVal

	err = sess.SaveDepth(context.Background(), aVal, 2)
	if err != nil {
		panic(err)
	}

	//load the object we just made (save will set the uuid)
	var readin VertexA
	err = sess.Load(context.Background(), &readin, aVal.UUID)
	if err != nil {
		panic(err)
	}

	fmt.Printf("%+v", readin)
}


```

## Migrating from V1 to V2

### Initialization
Initialization in gogm v1
```go
config := gogm.Config{
    IndexStrategy: gogm.VALIDATE_INDEX, //other options are ASSERT_INDEX and IGNORE_INDEX
    PoolSize:      50,
    Port:          7687,
    IsCluster:     false, //tells it whether or not to use `bolt+routing`
    Host:          "0.0.0.0",
    Password:      "password",
    Username:      "neo4j",
}

err := gogm.Init(&config, &VertexA{}, &VertexB{}, &EdgeC{})
if err != nil {
    panic(err)
}
```

Equivalent in GoGM v2
```go
// define your configuration
config := gogm.Config{
    IndexStrategy: gogm.VALIDATE_INDEX, //other options are ASSERT_INDEX and IGNORE_INDEX
    PoolSize:      50,
    Port:          7687,
    IsCluster:     false, //tells it whether or not to use `bolt+routing`
    Host:          "0.0.0.0",
    Password:      "password",
    Username:      "neo4j",
}

	// register all vertices and edges
	// this is so that GoGM doesn't have to do reflect processing of each edge in real time
	// use nil or gogm.DefaultPrimaryKeyStrategy if you only want graph ids
	_gogm, err := gogm.New(&config, gogm.UUIDPrimaryKeyStrategy, &VertexA{}, &VertexB{}, &EdgeC{})
	if err != nil {
		panic(err)
	}
	
	gogm.SetGlobalGoGM(_gogm)
```


##### Note that we call gogm.SetGloablGogm so that we can still access it from a package level

### Create a session
Creating a session in GoGM v1
```go
sess, err := gogm.NewSession(false)
```
##### Note this still works in v2, its using the global gogm to create the session. Also note this is making an instance of the deprecated `ISession`
Equivalent in GoGM v2
```go
// this would also work with a local instance of gogm (localGogm.NewSessionV2)
sess, err := gogm.G().NewSessionV2(gogm.SessionConfig{AccessMode: gogm.AccessModeWrite})
```

### Summary
- Minimal change requires creating a global gogm, everything else should still work with ISession (gogm v1 session object)
- `ISession` is now deprecated but still supported
- SessionV2 is the new standard
### GoGM CLI

## CLI Installation
```
go get -u github.com/ohbonsai/gogm/v2/cli/gogmcli
```

## CLI Usage
```
NAME:
   gogmcli - used for neo4j operations from gogm schema

USAGE:
   gogmcli [global options] command [command options] [arguments...]

VERSION:
   2.0.0

COMMANDS:
   generate, g, gen  to generate link and unlink functions for nodes
   help, h           Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --debug, -d    execute in debug mode (default: false)
   --help, -h     show help (default: false)
   --version, -v  print the version (default: false)
```

## Inspiration
Inspiration came from the Java OGM implementation by Neo4j.

## Road Map
- Schema Migration
- Errors overhaul using go 1.13 error wrapping

## How you can help
- Report Bugs
- Fix bugs
- Contribute (refer to contribute.md)
