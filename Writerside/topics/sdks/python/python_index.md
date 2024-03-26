# The SpacetimeDB Python client SDK

The SpacetimeDB client SDK for Python contains all the tools you need to build native clients for SpacetimeDB modules using Python.

## Install the SDK

Use pip to install the SDK:

```bash
pip install spacetimedb-sdk
```

## Generate module bindings

Each SpacetimeDB client depends on some bindings specific to your module. Create a `module_bindings` directory in your project's directory and generate the Python interface files using the Spacetime CLI. From your project directory, run:

```bash
mkdir -p module_bindings
spacetime generate --lang python \
    --out-dir module_bindings \
    --project-path PATH-TO-MODULE-DIRECTORY
```

Replace `PATH-TO-MODULE-DIRECTORY` with the path to your SpacetimeDB module.

Import your bindings in your client's code:

```python
import module_bindings
```

## Basic vs Async SpacetimeDB Client

This SDK provides two different client modules for interacting with your SpacetimeDB module.

The Basic client allows you to have control of the main loop of your application and you are responsible for regularly calling the client's `update` function. This is useful in settings like PyGame where you want to have full control of the main loop.

The Async client has a run function that you call after you set up all your callbacks and it will take over the main loop and handle updating the client for you. With the async client, you can have a regular "tick" function by using the `schedule_event` function.

## Common Client Reference

The following functions and types are used in both the Basic and Async clients.

### API at a glance

| Definition                                                                                              | Description                                                                                  |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Type [`Identity`](#type-identity.)                                                                       | A unique public identifier for a client.                                                     |
| Type [`Address`](#type-address.)                                                                         | An opaque identifier for differentiating connections by the same `Identity`.                 |
| Type [`ReducerEvent`](#type-reducerevent.)                                                               | `class` containing information about the reducer that triggered a row update event.          |
| Type [`module_bindings::{TABLE}`](#type-table.)                                                          | Autogenerated `class` type for a table, holding one row.                                     |
| Method [`module_bindings::{TABLE}::filter_by_{COLUMN}`](#method-filter_by_column.)                       | Autogenerated method to iterate over or seek subscribed rows where a column matches a value. |
| Method [`module_bindings::{TABLE}::iter`](#method-iter.)                                                 | Autogenerated method to iterate over all subscribed rows.                                    |
| Method [`module_bindings::{TABLE}::register_row_update`](#method-register_row_update.)                   | Autogenerated method to register a callback that fires when a row changes.                   |
| Function [`module_bindings::{REDUCER_NAME}::{REDUCER_NAME}`](#function-reducer.)                         | Autogenerated function to invoke a reducer.                                                  |
| Function [`module_bindings::{REDUCER_NAME}::register_on_{REDUCER_NAME}`](#function-register_on_reducer.) | Autogenerated function to register a callback to run whenever the reducer is invoked.        |

### Type `Identity`

```python
class Identity:
    @staticmethod
    def from_string(string)

    @staticmethod
    def from_bytes(data)

    def __str__(self)

    def __eq__(self, other)
```

| Member        | Args       | Meaning                              |
| ------------- | ---------- | ------------------------------------ |
| `from_string` | `str`      | Create an Identity from a hex string |
| `from_bytes`  | `bytes`    | Create an Identity from raw bytes    |
| `__str__`     | `None`     | Convert the Identity to a hex string |
| `__eq__`      | `Identity` | Compare two Identities for equality  |

A unique public identifier for a user of a database.

### Type `Address`

```python
class Address:
    @staticmethod
    def from_string(string)

    @staticmethod
    def from_bytes(data)

    def __str__(self)

    def __eq__(self, other)
```

| Member        | Type      | Meaning                             |
|---------------|-----------|-------------------------------------|
| `from_string` | `str`     | Create an Address from a hex string |
| `from_bytes`  | `bytes`   | Create an Address from raw bytes    |
| `__str__`     | `None`    | Convert the Address to a hex string |
| `__eq__`      | `Address` | Compare two Identities for equality |

An opaque identifier for a client connection to a database, intended to differentiate between connections from the same [`Identity`](#type-identity.).

### Type `ReducerEvent`

```python
class ReducerEvent:
    def __init__(self, caller_identity, reducer_name, status, message, args):
        self.caller_identity = caller_identity
        self.reducer_name = reducer_name
        self.status = status
        self.message = message
        self.args = args
```

| Member            | Type                | Meaning                                                                            |
|-------------------|---------------------|------------------------------------------------------------------------------------|
| `caller_identity` | `Identity`          | The identity of the user who invoked the reducer                                   |
| `caller_address`  | `Optional[Address]` | The address of the user who invoked the reducer, or `None` for scheduled reducers. |
| `reducer_name`    | `str`               | The name of the reducer that was invoked                                           |
| `status`          | `str`               | The status of the reducer invocation ("committed", "failed", "outofenergy")        |
| `message`         | `str`               | The message returned by the reducer if it fails                                    |
| `args`            | `List[str]`         | The arguments passed to the reducer                                                |

This class contains the information about a reducer event to be passed to row update callbacks.

### Type `{TABLE}`

```python
class TABLE:
	is_table_class = True

	primary_key = "identity"

	@classmethod
	def register_row_update(cls, callback: Callable[[str,TABLE,TABLE,ReducerEvent], None])

	@classmethod
	def iter(cls) -> Iterator[User]

	@classmethod
	def filter_by_COLUMN_NAME(cls, COLUMN_VALUE) -> TABLE
```

This class is autogenerated for each table in your module. It contains methods for filtering and iterating over subscribed rows.

### Method `filter_by_{COLUMN}`

```python
def filter_by_COLUMN(self, COLUMN_VALUE) -> TABLE
```

| Argument       | Type          | Meaning                |
| -------------- | ------------- | ---------------------- |
| `column_value` | `COLUMN_TYPE` | The value to filter by |

For each column of a table, `spacetime generate` generates a `classmethod` on the [table class](#type-table.) to filter or seek subscribed rows where that column matches a requested value. These methods are named `filter_by_{COLUMN}`, where `{COLUMN}` is the column name converted to `snake_case`.

The method's return type depends on the column's attributes:

- For unique columns, including those annotated `#[unique]` and `#[primarykey]`, the `filter_by` method returns a `{TABLE}` or None, where `{TABLE}` is the [table struct](#type-table.).
- For non-unique columns, the `filter_by` method returns an `Iterator` that can be used in a `for` loop.

### Method `iter`

```python
def iter(self) -> Iterator[TABLE]
```

Iterate over all the subscribed rows in the table.

### Method `register_row_update`

```python
def register_row_update(self, callback: Callable[[str,TABLE,TABLE,ReducerEvent], None])
```

| Argument   | Type                                      | Meaning                                                                                          |
| ---------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `callback` | `Callable[[str,TABLE,TABLE,ReducerEvent]` | Callback to be invoked when a row is updated (Args: row_op, old_value, new_value, reducer_event) |

Register a callback function to be executed when a row is updated. Callback arguments are:

- `row_op`: The type of row update event. One of `"insert"`, `"delete"`, or `"update"`.
- `old_value`: The previous value of the row, `None` if the row was inserted.
- `new_value`: The new value of the row, `None` if the row was deleted.
- `reducer_event`: The [`ReducerEvent`](#type-reducerevent.) that caused the row update, or `None` if the row was updated as a result of a subscription change.

### Function `{REDUCER_NAME}`

```python
def {REDUCER_NAME}(arg1, arg2)
```

This function is autogenerated for each reducer in your module. It is used to invoke the reducer. The arguments match the arguments defined in the reducer's `#[reducer]` attribute.

### Function `register_on_{REDUCER_NAME}`

```python
def register_on_{REDUCER_NAME}(callback: Callable[[Identity, Optional[Address], str, str, ARG1_TYPE, ARG1_TYPE], None])
```

| Argument   | Type                                                         | Meaning                                                                                           |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `callback` | `Callable[[Identity, str, str, ARG1_TYPE, ARG1_TYPE], None]` | Callback to be invoked when the reducer is invoked (Args: caller_identity, status, message, args) |

Register a callback function to be executed when the reducer is invoked. Callback arguments are:

- `caller_identity`: The identity of the user who invoked the reducer.
- `caller_address`: The address of the user who invoked the reducer, or `None` for scheduled reducers.
- `status`: The status of the reducer invocation ("committed", "failed", "outofenergy").
- `message`: The message returned by the reducer if it fails.
- `args`: Variable number of arguments passed to the reducer.

## Async Client Reference

### API at a glance

| Definition                                                                                                        | Description                                                                                              |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Function [`SpacetimeDBAsyncClient::run`](#function-run.)                                                           | Run the client. This function will not return until the client is closed.                                |
| Function [`SpacetimeDBAsyncClient::subscribe`](#function-subscribe.)                                               | Subscribe to receive data and transaction updates for the provided queries.                              |
| Function [`SpacetimeDBAsyncClient::register_on_subscription_applied`](#function-register_on_subscription_applied.) | Register a callback when the local cache is updated as a result of a change to the subscription queries. |
| Function [`SpacetimeDBAsyncClient::force_close`](#function-force_close.)                                           | Signal the client to stop processing events and close the connection to the server.                      |
| Function [`SpacetimeDBAsyncClient::schedule_event`](#function-schedule_event.)                                     | Schedule an event to be fired after a delay                                                              |

### Function `run`

```python
async def run(
        self,
        auth_token,
        host,
        address_or_name,
        ssl_enabled,
        on_connect,
        subscription_queries=[],
    )
```

Run the client. This function will not return until the client is closed.

| Argument               | Type                              | Meaning                                                        |
| ---------------------- | --------------------------------- | -------------------------------------------------------------- |
| `auth_token`           | `str`                             | Auth token to authenticate the user. (None if new user)        |
| `host`                 | `str`                             | Hostname of SpacetimeDB server                                 |
| `address_or_name`      | `&str`                            | Name or address of the module.                                 |
| `ssl_enabled`          | `bool`                            | Whether to use SSL when connecting to the server.              |
| `on_connect`           | `Callable[[str, Identity], None]` | Callback to be invoked when the client connects to the server. |
| `subscription_queries` | `List[str]`                       | List of queries to subscribe to.                               |

If `auth_token` is not None, they will be passed to the new connection to identify and authenticate the user. Otherwise, a new Identity and auth token will be generated by the server. An optional [local_config](#local_config.) module can be used to store the user's auth token to local storage.

If you are connecting to SpacetimeDB Cloud `testnet` the host should be `testnet.spacetimedb.com` and `ssl_enabled` should be `True`. If you are connecting to SpacetimeDB Standalone locally, the host should be `localhost:3000` and `ssl_enabled` should be `False`. For instructions on how to deploy to these environments, see the [Deployment Section](testnet.)

```python
asyncio.run(
    spacetime_client.run(
        AUTH_TOKEN,
        "localhost:3000",
        "my-module-name",
        False,
        on_connect,
        ["SELECT * FROM User", "SELECT * FROM Message"],
    )
)
```

### Function `subscribe`

```rust
def subscribe(self, queries: List[str])
```

Subscribe to a set of queries, to be notified when rows which match those queries are altered.

| Argument  | Type        | Meaning                      |
| --------- | ----------- | ---------------------------- |
| `queries` | `List[str]` | SQL queries to subscribe to. |

The `queries` should be a slice of strings representing SQL queries.

A new call to `subscribe` will remove all previous subscriptions and replace them with the new `queries`. If any rows matched the previous subscribed queries but do not match the new queries, those rows will be removed from the client cache. Row update events will be dispatched for any inserts and deletes that occur as a result of the new queries. For these events, the [`ReducerEvent`](#type-reducerevent.) argument will be `None`.

This should be called before the async client is started with [`run`](#function-run.).

```python
spacetime_client.subscribe(["SELECT * FROM User;", "SELECT * FROM Message;"])
```

Subscribe to a set of queries, to be notified when rows which match those queries are altered.

### Function `register_on_subscription_applied`

```python
def register_on_subscription_applied(self, callback)
```

Register a callback function to be executed when the local cache is updated as a result of a change to the subscription queries.

| Argument   | Type                 | Meaning                                                |
| ---------- | -------------------- | ------------------------------------------------------ |
| `callback` | `Callable[[], None]` | Callback to be invoked when subscriptions are applied. |

The callback will be invoked after a successful [`subscribe`](#function-subscribe.) call when the initial set of matching rows becomes available.

```python
spacetime_client.register_on_subscription_applied(on_subscription_applied)
```

### Function `force_close`

```python
def force_close(self)
)
```

Signal the client to stop processing events and close the connection to the server.

```python
spacetime_client.force_close()
```

### Function `schedule_event`

```python
def schedule_event(self, delay_secs, callback, *args)
```

Schedule an event to be fired after a delay

To create a repeating event, call schedule_event() again from within the callback function.

| Argument     | Type                 | Meaning                                                        |
| ------------ | -------------------- | -------------------------------------------------------------- |
| `delay_secs` | `float`              | number of seconds to wait before firing the event              |
| `callback`   | `Callable[[], None]` | Callback to be invoked when the event fires.                   |
| `args`       | `*args`              | Variable number of arguments to pass to the callback function. |

```python
def application_tick():
    # ... do some work

    spacetime_client.schedule_event(0.1, application_tick)

spacetime_client.schedule_event(0.1, application_tick)
```

## Basic Client Reference

### API at a glance

| Definition                                                                                                       | Description                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Function [`SpacetimeDBClient::init`](#function-init.)                                                             | Create a network manager instance.                                                                                               |
| Function [`SpacetimeDBClient::subscribe`](#function-subscribe.)                                                   | Subscribe to receive data and transaction updates for the provided queries.                                                      |
| Function [`SpacetimeDBClient::register_on_event`](#function-register_on_event.)                                   | Register a callback function to handle transaction update events.                                                                |
| Function [`SpacetimeDBClient::unregister_on_event`](#function-unregister_on_event.)                               | Unregister a callback function that was previously registered using `register_on_event`.                                         |
| Function [`SpacetimeDBClient::register_on_subscription_applied`](#function-register_on_subscription_applied.)     | Register a callback function to be executed when the local cache is updated as a result of a change to the subscription queries. |
| Function [`SpacetimeDBClient::unregister_on_subscription_applied`](#function-unregister_on_subscription_applied.) | Unregister a callback function from the subscription update event.                                                               |
| Function [`SpacetimeDBClient::update`](#function-update.)                                                         | Process all pending incoming messages from the SpacetimeDB module.                                                               |
| Function [`SpacetimeDBClient::close`](#function-close.)                                                           | Close the WebSocket connection.                                                                                                  |
| Type [`TransactionUpdateMessage`](#type-transactionupdatemessage.)                                                | Represents a transaction update message.                                                                                         |

### Function `init`

```python
@classmethod
def init(
    auth_token: str,
    host: str,
    address_or_name: str,
    ssl_enabled: bool,
    autogen_package: module,
    on_connect: Callable[[], NoneType] = None,
    on_disconnect: Callable[[str], NoneType] = None,
    on_identity: Callable[[str, Identity, Address], NoneType] = None,
    on_error: Callable[[str], NoneType] = None
)
```

Create a network manager instance.

| Argument          | Type                                       | Meaning                                                                                                                                                                                              |
|-------------------|--------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `auth_token`      | `str`                                      | This is the token generated by SpacetimeDB that matches the user's identity. If None, token will be generated                                                                                        |
| `host`            | `str`                                      | Hostname:port for SpacetimeDB connection                                                                                                                                                             |
| `address_or_name` | `str`                                      | The name or address of the database to connect to                                                                                                                                                    |
| `ssl_enabled`     | `bool`                                     | Whether to use SSL when connecting to the server.                                                                                                                                                    |
| `autogen_package` | `ModuleType`                               | Python package where SpacetimeDB module generated files are located.                                                                                                                                 |
| `on_connect`      | `Callable[[], None]`                       | Optional callback called when a connection is made to the SpacetimeDB module.                                                                                                                        |
| `on_disconnect`   | `Callable[[str], None]`                    | Optional callback called when the Python client is disconnected from the SpacetimeDB module. The argument is the close message.                                                                      |
| `on_identity`     | `Callable[[str, Identity, Address], None]` | Called when the user identity is recieved from SpacetimeDB. First argument is the auth token used to login in future sessions. Third argument is the client connection's [`Address`](#type-address.). |
| `on_error`        | `Callable[[str], None]`                    | Optional callback called when the Python client connection encounters an error. The argument is the error message.                                                                                   |

This function creates a new SpacetimeDBClient instance. It should be called before any other functions in the SpacetimeDBClient class. This init will call connect for you.

```python
SpacetimeDBClient.init(autogen, on_connect=self.on_connect)
```

### Function `subscribe`

```python
def subscribe(queries: List[str])
```

Subscribe to receive data and transaction updates for the provided queries.

| Argument  | Type        | Meaning                                                                                                  |
| --------- | ----------- | -------------------------------------------------------------------------------------------------------- |
| `queries` | `List[str]` | A list of queries to subscribe to. Each query is a string representing an sql formatted query statement. |

This function sends a subscription request to the SpacetimeDB module, indicating that the client wants to receive data and transaction updates related to the specified queries.

```python
queries = ["SELECT * FROM table1", "SELECT * FROM table2 WHERE col2 = 0"]
SpacetimeDBClient.instance.subscribe(queries)
```

### Function `register_on_event`

```python
def register_on_event(callback: Callable[[TransactionUpdateMessage], NoneType])
```

Register a callback function to handle transaction update events.

| Argument   | Type                                         | Meaning                                                                                                                                                                                                                  |
| ---------- | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `callback` | `Callable[[TransactionUpdateMessage], None]` | A callback function that takes a single argument of type `TransactionUpdateMessage`. This function will be invoked with a `TransactionUpdateMessage` instance containing information about the transaction update event. |

This function registers a callback function that will be called when a reducer modifies a table matching any of the subscribed queries or if a reducer called by this Python client encounters a failure.

```python
def handle_event(transaction_update):
    # Code to handle the transaction update event

SpacetimeDBClient.instance.register_on_event(handle_event)
```

### Function `unregister_on_event`

```python
def unregister_on_event(callback: Callable[[TransactionUpdateMessage], NoneType])
```

Unregister a callback function that was previously registered using `register_on_event`.

| Argument   | Type                                         | Meaning                              |
| ---------- | -------------------------------------------- | ------------------------------------ |
| `callback` | `Callable[[TransactionUpdateMessage], None]` | The callback function to unregister. |

```python
SpacetimeDBClient.instance.unregister_on_event(handle_event)
```

### Function `register_on_subscription_applied`

```python
def register_on_subscription_applied(callback: Callable[[], NoneType])
```

Register a callback function to be executed when the local cache is updated as a result of a change to the subscription queries.

| Argument   | Type                 | Meaning                                                                                                                                                      |
| ---------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `callback` | `Callable[[], None]` | A callback function that will be invoked on each subscription update. The callback function should not accept any arguments and should not return any value. |

```python
def subscription_callback():
    # Code to be executed on each subscription update

SpacetimeDBClient.instance.register_on_subscription_applied(subscription_callback)
```

### Function `unregister_on_subscription_applied`

```python
def unregister_on_subscription_applied(callback: Callable[[], NoneType])
```

Unregister a callback function from the subscription update event.

| Argument   | Type                 | Meaning                                                                                                  |
| ---------- | -------------------- | -------------------------------------------------------------------------------------------------------- |
| `callback` | `Callable[[], None]` | A callback function that was previously registered with the `register_on_subscription_applied` function. |

```python
def subscription_callback():
    # Code to be executed on each subscription update

SpacetimeDBClient.instance.register_on_subscription_applied(subscription_callback)
```

### Function `update`

```python
def update()
```

Process all pending incoming messages from the SpacetimeDB module.

This function must be called on a regular interval in the main loop to process incoming messages.

```python
while True:
    SpacetimeDBClient.instance.update() # Call the update function in a loop to process incoming messages
    # Additional logic or code can be added here
```

### Function `close`

```python
def close()
```

Close the WebSocket connection.

This function closes the WebSocket connection to the SpacetimeDB module.

```python
SpacetimeDBClient.instance.close()
```

### Type `TransactionUpdateMessage`

```python
class TransactionUpdateMessage:
    def __init__(
        self,
        caller_identity: Identity,
        status: str,
        message: str,
        reducer_name: str,
        args: Dict
    )
```

| Member            | Args       | Meaning                                           |
| ----------------- | ---------- | ------------------------------------------------- |
| `caller_identity` | `Identity` | The identity of the caller.                       |
| `status`          | `str`      | The status of the transaction.                    |
| `message`         | `str`      | A message associated with the transaction update. |
| `reducer_name`    | `str`      | The reducer used for the transaction.             |
| `args`            | `Dict`     | Additional arguments for the transaction.         |

Represents a transaction update message. Used in on_event callbacks.

For more details, see [`register_on_event`](#function-register_on_event.).