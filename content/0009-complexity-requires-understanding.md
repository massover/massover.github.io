Title: Complexity requires understanding
Date: 2022-01-07 11:05
Category: python

I generate yaml files for [Open API Schemas](https://swagger.io/specification/). These Open API schemas are often dynamically
generated from something like [django rest framework](https://www.django-rest-framework.org/api-guide/schemas/#generating-an-openapi-schema).
Deep in the [schema code](https://github.com/encode/django-rest-framework/blob/master/rest_framework/schemas/openapi.py#L119), I
was adding an `example` such that a fixture would appear in the documentation. This resulted in a yaml schema that 
was not renderable by [Redoc](https://github.com/Redocly/redoc).

```python
import uuid
from rest_framework.schemas.openapi import SchemaGenerator

class MySchemaGenerator(SchemaGenerator):
    def get_responses(self, path, method):
        responses = super().get_responses(path, method)
    
        if "fixture" in path:
            for key, value in responses["200"]["content"].items():
                value["schema"] = {
                    "type": "object",
                    "items": {},
                    "example": {"id": uuid.uuid4()},
                    "description": "Use the example for the schema structure.",
                }
        return responses
```

A `UUID` is serializable in yaml, but not in a helpful way when generating an open api schema.

```python
>>> import yaml
>>> import uuid
>>> d = {"id": uuid.uuid4()}
>>> yaml.dump(d)
'id: !!python/object:uuid.UUID\n  int: 132234147215218020088149232009421531513\n'
```

This serialization problem exists with `json` as well although json raises a runtime error.

```python
>>> import json
>>> json.dumps(d)
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-8-1a41a0b88650> in <module>
----> 1 json.dumps(d)

...

TypeError: Object of type UUID is not JSON serializable
```

This problem is solved for `json` by creating a custom `json.JSONEncoder` that can handle complex types

```python
import json
class MyJSONEncoder(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, uuid.UUID):
            return str(o)
        return super().default(o)
```

```python

>>> json.dumps(d, cls=MyJSONEncoder)
>>> '{"id": "8a743995-54d6-46bd-bc37-39ce46558b46"}'
```

My initial thoughts about the same for `yaml` were that it was too complex. I needed to learn about 
`dumpers`, `representers`, `scalers` and `nodes` just to call `str` around some value that needed to be serialized.

```python
import yaml

def uuid_representer(dumper, data):
    return dumper.represent_str(str(data))

yaml.add_representer(uuid.UUID, uuid_representer)
```

```python
>>> result = yaml.dump(id)
>>> result
'id: 637b5ed1-2c57-47d1-aa28-9d586f902979\n'
>>> yaml.load(result)
 {'id': '637b5ed1-2c57-47d1-aa28-9d586f902979'}
```

`yaml` also has the ability to encode the context required for complex object deserialization.

```python
import uuid
import yaml


def uuid_representer(dumper, data):
    return dumper.represent_scalar('!uuid', str(data))

def uuid_constructor(loader, node):
     value = loader.construct_scalar(node)
     return uuid.UUID(value)


yaml.add_representer(uuid.UUID, uuid_representer)
yaml.add_constructor('!uuid', uuid_constructor)
```

```python
>>> id = {"id": uuid.uuid4()}
>>> result = yaml.dump(id)
>>> result
"id: !uuid '2c3b217e-c52f-4af5-9190-ed42f1274945'\n"
>>> yaml.load(result)
{'id': UUID('2c3b217e-c52f-4af5-9190-ed42f1274945')}
```

Unfortunately, it doesn't have a benefit for open api. Since `uuid` is not part of the open api spec, none of the clients
such as `Redoc` would know how to deserialize my custom `uuid` encoding. 

While my first thought was that it was too complex to perform the uuid serialization in `yaml`, now I understand some 
more of the tradeoffs in the spec which drive the implementation. Have you ever misjudged complexity from a lack of understanding?