# Writing a module

A module is a Python class that uses Synapse's module API to interact with the
homeserver. It can register callbacks that Synapse will call on specific operations, as
well as web resources to attach to Synapse's web server.

When instantiated, a module is given its parsed configuration as well as an instance of
the `synapse.module_api.ModuleApi` class. The configuration is a dictionary, and is
either the output of the module's `parse_config` static method (see below), or the
configuration associated with the module in Synapse's configuration file.

See the documentation for the `ModuleApi` class
[here](https://github.com/matrix-org/synapse/blob/master/synapse/module_api/__init__.py).

## Handling the module's configuration

A module can implement the following static method:

```python
@staticmethod
def parse_config(config: dict) -> dict
```

This method is given a dictionary resulting from parsing the YAML configuration for the
module. It may modify it (for example by parsing durations expressed as strings (e.g.
"5d") into milliseconds, etc.), and return the modified dictionary. It may also verify
that the configuration is correct, and raise an instance of
`synapse.module_api.errors.ConfigError` if not.

## Registering a web resource

Modules can register web resources onto Synapse's web server using the following module
API method:

```python
def ModuleApi.register_web_resource(path: str, resource: IResource) -> None
```

The path is the full absolute path to register the resource at. For example, if you
register a resource for the path `/_synapse/client/my_super_module/say_hello`, Synapse
will serve it at `http(s)://[HS_URL]/_synapse/client/my_super_module/say_hello`. Note
that Synapse does not allow registering resources for several sub-paths in the `/_matrix`
namespace (such as anything under `/_matrix/client` for example). It is strongly
recommended that modules register their web resources under the `/_synapse/client`
namespace.

The provided resource is a Python class that implements Twisted's [IResource](https://twistedmatrix.com/documents/current/api/twisted.web.resource.IResource.html)
interface (such as [Resource](https://twistedmatrix.com/documents/current/api/twisted.web.resource.Resource.html)).

Only one resource can be registered for a given path. If several modules attempt to
register a resource for the same path, the module that appears first in Synapse's
configuration file takes priority.

Modules **must** register their web resources in their `__init__` method.

## Registering a callback

Modules can use Synapse's module API to register callbacks. Callbacks are functions that
Synapse will call when performing specific actions. Callbacks must be asynchronous, and
are split in categories. A single module may implement callbacks from multiple categories,
and is under no obligation to implement all callbacks from the categories it registers
callbacks for.

Modules can register callbacks using one of the module API's `register_[...]_callbacks`
methods. The callback functions are passed to these methods as keyword arguments, with
the callback name as the argument name and the function as its value. This is demonstrated
in the example below. A `register_[...]_callbacks` method exists for each category.


## Example

The example below is a module that implements the spam checker callback
`user_may_create_room` to deny room creation to user `@evilguy:example.com`, and registers
a web resource to the path `/_synapse/client/demo/hello` that returns a JSON object.

```python
import json

from twisted.web.resource import Resource
from twisted.web.server import Request

from synapse.module_api import ModuleApi


class DemoResource(Resource):
    def __init__(self, config):
        super(DemoResource, self).__init__()
        self.config = config

    def render_GET(self, request: Request):
        name = request.args.get(b"name")[0]
        request.setHeader(b"Content-Type", b"application/json")
        return json.dumps({"hello": name})


class DemoModule:
    def __init__(self, config: dict, api: ModuleApi):
        self.config = config
        self.api = api

        self.api.register_web_resource(
            path="/_synapse/client/demo/hello",
            resource=DemoResource(self.config),
        )

        self.api.register_spam_checker_callbacks(
            user_may_create_room=self.user_may_create_room,
        )

    @staticmethod
    def parse_config(config):
        return config

    async def user_may_create_room(self, user: str) -> bool:
        if user == "@evilguy:example.com":
            return False

        return True
```