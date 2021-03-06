+++
date = "2017-03-20T19:35:35+11:00"
title = "Clients"
+++

## Implementation

Clients can communicate with the server in two different ways:

- **Via [gRPC](http://www.grpc.io/).** Internally this uses [Protocol
  Buffers](https://developers.google.com/protocol-buffers) (the proto file
used by Dgraph is located at
[task.proto](https://github.com/dgraph-io/dgraph/blob/master/protos/task.proto)).

- **Via HTTP.** There are various endpoints, each accepting and returning JSON.
  There is a one to one correspondence between the HTTP endpoints and the gRPC
service methods.


It's possible to interface with dgraph directly via gRPC or HTTP. However, if a
client library exists for you language, this will be an easier option.

## Go

[![GoDoc](https://godoc.org/github.com/dgraph-io/dgraph/client?status.svg)](https://godoc.org/github.com/dgraph-io/dgraph/client)

The go client communicates with the server on the grpc port (default value 9080).

The client can be obtained in the usual way via `go get`:

```sh
go get -u -v github.com/dgraph-io/dgraph/client
```

The full [GoDoc](https://godoc.org/github.com/dgraph-io/dgraph/client) contains
documentation for the client API along with examples showing how to use it.

### Create the client

To create a client, dial a connection to Dgraph's external Grpc port (typically
9080). The following code snippet shows just one connection. You can connect to multiple Dgraph servers to distribute the workload evenly.

```go
func newClient() *client.Dgraph {
	// Dial a gRPC connection. The address to dial to can be configured when
	// setting up the dgraph cluster.
	d, err := grpc.Dial("localhost:9080", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}

	return client.NewDgraphClient(
		protos.NewDgraphClient(d),
	)
}
```

### Alter the database

To set the schema, set it on a `protos.Operation` object, and pass it down to
the `Alter` method.

```go
func setup(c *client.Dgraph) {
	// Install a schema into dgraph. Accounts have a `name` and a `balance`.
	err := c.Alter(ctx, &protos.Operation{
		Schema: `
			name: string @index(term) .
			balance: int .
		`,
	})
}
```

`protos.Operation` contains other fields as well, including drop predicate and
drop all. Drop all is useful if you wish to discard all the data, and start from
a clean slate, without bringing the instance down.

```go
	// Drop all data including schema from the dgraph instance. This is useful
	// for small examples such as this, since it puts dgraph into a clean
	// state.
	err := c.Alter(ctx, &protos.Operation{DropAll: true})
```

### Create a transaction

Dgraph v0.9 supports running distributed ACID transactions. To create a
transaction, just call `c.NewTxn()`. This operation incurs no network call.
Typically, you'd also want to call a `defer txn.Discard()` to let it
automatically rollback in case of errors. Calling `Discard` after `Commit` would
be a no-op.

```go
func runTxn(c *client.Dgraph) {
	txn := c.NewTxn()
	defer txn.Discard()
	...
}
```

### Run a query

You can run a query by calling `txn.Query`. The response would contain a `JSON`
field, which has the JSON encoded result. You can unmarshal it into Go struct
via `json.Unmarshal`.

```go
	// Query the balance for Alice and Bob.
	const q = `
		{
			all(func: anyofterms(name, "Alice Bob")) {
				uid
				balance
			}
		}
	`
	resp, err := txn.Query(context.Background(), q)
	if err != nil {
		log.Fatal(err)
	}

	// After we get the balances, we have to decode them into structs so that
	// we can manipulate the data.
	var decode struct {
		All []struct {
			Uid     string
			Balance int
		}
	}
	if err := json.Unmarshal(resp.GetJson(), &decode); err != nil {
		log.Fatal(err)
	}
```

### Run a mutation

`txn.Mutate` would run the mutation. It takes in a `protos.Mutation` object,
which provides two main ways to set data: JSON and RDF N-Quad. You can choose
whichever way is convenient.

We're going to continue using JSON. You could modify the Go structs parsed from
the query, and marshal them back into JSON.

```go
	// Move $5 between the two accounts.
	decode.All[0].Bal += 5
	decode.All[1].Bal -= 5

	out, err := json.Marshal(decode.All)
	if err != nil {
		log.Fatal(err)
	}

	_, err := txn.Mutate(ctx, &protos.Mutation{SetJSON: out})
```

Sometimes, you only want to commit mutation, without querying anything further.
In such cases, you can use a `CommitNow` field in `protos.Mutation` to
indicate that the mutation must be immediately committed.

### Commit the transaction

Once all the queries and mutations are done, you can commit the transaction. It
returns an error in case the transaction could not be committed.

```go
	// Finally, we can commit the transactions. An error will be returned if
	// other transactions running concurrently modify the same data that was
	// modified in this transaction. It is up to the library user to retry
	// transactions when they fail.

	err := txn.Commit(ctx)
```

## Java

The Java client is a new and fully supported client for v0.9.0.

The client can be found [here](https://github.com/dgraph-io/dgraph4j).

## Javascript

{{% notice "note" %}}
A Javascript client doesn't exist yet. But due to popular demand, a Javascript
client will be created to work with dgraph v0.9.0. Watch this space!
{{% /notice %}}

## Python
{{% notice "incomplete" %}}
A lot of development has gone into the Go client and the Python client is not up to date with it.
The Python client is not compatible with dgraph v0.9.0 and onwards.
We are looking for help from contributors to bring it up to date.
{{% /notice %}}

The Python client can be found [here](https://github.com/dgraph-io/pydgraph).

## Raw HTTP

It's also possible to interact with dgraph directly via its HTTP endpoints.
This this allows clients to be built for languages that don't have access to a
working gRPC implementation.

In the examples shown here, regular command line tools such as `curl` and
[`jq`](https://stedolan.github.io/jq/) are used. However, the real intention
here is to show other programmers how they could implement a client in their
language on top of the HTTP API.

Similar to the Go client example, we use a bank account transfer example.

### Create the Client

A client build on top of the HTTP API will need to track state at two different
levels:

1. Per client. Each client will need to keep a linearized reads (`lin_read`)
   map. This is a map from dgraph group id to proposal id. This will be needed
for the system as a whole (client + server) to have
[linearizability](https://en.wikipedia.org/wiki/Linearizability). Whenever a
`lin_read` map is received, the client should update its version of the map by
merging the two maps together. The merge operation is simple - the new map gets
all key/value pairs from the parent maps. Where a key exists in both maps, the
max value is taken.

2. Per transaction. There are three pieces of state that need to be maintained
   for each transaction.

    1. Each transaction needs its own `lin_read` (updated independently of the
       client level `lin_read`).
  
    2. A start timestamp (`start_ts`). This uniquely identifies a transaction,
       and doesn't change over the transaction lifecycle.
  
    3. The set of keys modified by the transaction (`keys`). This aids in
       transaction conflict detection.

### Alter the database

The `/alter` endpoint is used to create or change the schema. Here, the
predicate `name` is the name of an account. It's indexed so that we can look up
accounts based on their name.

```sh
curl -X POST localhost:8080/alter -d 'name: string @index(term) .'
```

If all goes well, the response should be `{"code":"Success","message":"Done"}`.

Other operations can be performed via the `/alter` endpoint as well. A specific
predicate or the entire database can be dropped.

E.g. to drop the predicate `name`:
```sh
curl -X POST localhost:8080/alter -d '{"drop_attr": "name"}'
```
To drop all data and schema:
```sh
curl -X POST localhost:8080/alter -d '{"drop_all": true}'
```

### Start a transaction

Assume some initial accounts with balances have been populated. We now want to
transfer money from one account to the other. This is done in four steps:

1. Create a new transaction.

1. Inside the transaction, run a query to determine the current balances.

2. Perform a mutation to update the balances.

3. Commit the transaction.

Starting a transaction doesn't require any interaction with dgraph itself.
Some state needs to be set up for the transaction to use. `lin_read` is
initialized by *copying* the client's `lin_read`. The `start_ts` can initially
be set to 0. `keys` can start as an empty set.

### Run a query

To query the database, the `/query` endpoint is used. We need to use the
transaction scoped `lin_read`. Assume that `lin_read` is `{"1": 12}`.

To get the balances for both accounts:

```sh
curl -X POST -H 'X-Dgraph-LinRead: {"1": 12}' localhost:8080/query -d $'
{
  balances(func: anyofterms(name, "Alice Bob")) {
    uid
    name
    balance
  }
}' | jq

```

The result should look like this:

```json
{
  "data": {
    "balances": [
      {
        "uid": "0x1",
        "name": "Alice",
        "balance": "100"
      },
      {
        "uid": "0x2",
        "name": "Bob",
        "balance": "70"
      }
    ]
  },
  "extensions": {
    "server_latency": {
      "parsing_ns": 70494,
      "processing_ns": 697140,
      "encoding_ns": 1560151
    },
    "txn": {
      "start_ts": 4,
      "lin_read": {
        "ids": {
          "1": 14
        }
      }
    }
  }
}
```

Notice that along with the query result under the `data` field, there is some
additional data in the `extensions -> txn` field. This data will have to be
tracked by the client.

First, there is a `start_ts` in the response. This `start_ts` will need to be
used in all subsequent interactions with dgraph for this transaction, and so
should become part of the transaction state.

Second, there is a new `lin_read` map. The `lin_read` map should be merged with
both the client scoped and transaction scoped `lin_read` maps. Recall that both
the transaction scoped and client scoped `lin_read` maps are `{"1": 12}`. The
`lin_read` in the response is `{"1": 14}`. The merged result is `{"1": 14}`,
since we take the max all of the keys.

### Run a Mutation

Now that we have the current balances, we need to send a mutation to dgraph
with the updated balances. If Bob transfers $10 to Alice, then the RDFs to send
are:

```
<0x1> <balance> "110" .
<0x2> <balance> "60" .
```
Note that we have to to refer to the Alice and Bob nodes by UID in the RDF
format.

We now send the mutations via the `/mutate` endpoint. We need to provide our
transaction start timestamp via the header, so that dgraph knows which
transaction the mutation should be part of.

```sh
curl -X POST -H 'X-Dgraph-StartTs: 4' localhost:8080/mutate -d $'
{
  set {
    <0x1> <balance> "110" .
    <0x2> <balance> "60" .
  }
}
' | jq
```

The result:

```json
{
  "data": {
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {
    "txn": {
      "start_ts": 4,
      "keys": [
        "AAALX3ByZWRpY2F0ZV8AAAAAAAAAAAI=",
        "AAAHYmFsYW5jZQAAAAAAAAAAAg==",
        "AAALX3ByZWRpY2F0ZV8AAAAAAAAAAAE=",
        "AAAHYmFsYW5jZQAAAAAAAAAAAQ=="
      ],
      "lin_read": {
        "ids": {
          "1": 17
        }
      }
    }
  }
}
```

We get another `lin_read` map, which needs to be merged (the new `lin_read` map
for **both the client and transaction** becomes `{"1": 17}`). We also get some
`keys`. These should be added to the set of `keys` stored in the transaction
state.

### Committing the transaction

Finally, we can commit the transaction using the `/commit` endpoint. We need
the `start_ts` we've been using for the transaction along with the `keys`.
If we had performed multiple mutations in the transaction instead of the just
the one, then the keys provided during the commit would be the union of all
keys returned in the responses from the `/mutate` endpoint.

```sh
curl -X POST -H 'X-Dgraph-StartTs: 4' -H 'X-Dgraph-Keys: ["AAALX3ByZWRpY2F0ZV8AAAAAAAAAAAI=","AAAHYmFsYW5jZQAAAAAAAAAAAg==","AAALX3ByZWRpY2F0ZV8AAAAAAAAAAAE=","AAAHYmFsYW5jZQAAAAAAAAAAAQ=="]' localhost:8080/commit | jq
```

```json
{
  "data": {
    "code": "Success",
    "message": "Done"
  },
  "extensions": {
    "txn": {
      "start_ts": 4,
      "commit_ts": 5
    }
  }
}
```

Notice that we receive a `commit_ts` in the response (this field was previously
absent). The transaction is now complete.

If another client were to perform another transaction concurrently affecting
the same keys, then it's possible that the transaction would *not* be
successful.  This is indicated in the response when the commit is attempted.

```json
{
  "errors": [
    {
      "code": "Error",
      "message": "Transaction aborted"
    }
  ]
}
```

In this case, it should be up to the user of the client to decide if they wish
to retry the transaction.
