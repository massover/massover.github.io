Title: Logs don't lie 
Date: 2021-10-28 10:43
Category: python

Start with an innocuous log statement and the corresponding generated json formatted logs

```python
logger.info({"message": "Fetch user", "id": id})
```

```text
{"message": "Fetch user", "id": "e8f468ec-007c-4b36-86a9-34c7bdb72a5f"}
```

In isolation, is there any way to know what type `id` is? Unfortunately there really isn't. What's missing is how the logs get 
formatted. By default there is no json serialization for a UUID. UUIDs are usually serialized to strings.

```python
>>> json.dumps(uuid.uuid4())
Traceback (most recent call last):
    ...
TypeError: Object of type UUID is not JSON serializable
```

From just the log statement, we lose context about whether `id` is `"0d45a880-6c1f-4ad9-b0b9-2546e443706e"` or 
`UUID('0d45a880-6c1f-4ad9-b0b9-2546e443706e')`. When dealing with systems interacting with json, both of these
are going to be valid use cases. Any non default json serializable type (another example would be a timestamp) would 
suffer from this ambiguity.

The logs don't lie. Sometimes we do need to stretch the truth.






