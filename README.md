# Dynamic Serializers

## A tailored response with query parameters

So far, we've learned how to modify our response with our serializer in Django by setting the fields attribute and setting the depth to get nested data in foreign key relationships, but what if we wanted to be able to modify the response conditionally? Let's create a dynamic serializer that can be used on any model, and we will pass in query parameters to get the data formatted exactly how we want it!

## Creating a flexible serializer

Add this serializer to your code, you can put it in its own serializer or utility module if you like, that way you can pass it to any of your serializers easily.

```py
class DynamicModelSerializer(serializers.ModelSerializer):
    """
    A ModelSerializer that takes some additional arguments: `fields`, which
    controls which fields should be displayed, a `nest`
    argument to conditionally set the depth, and `exclude` to add any fields you would like excluded from the response
    """

    def __init__(self, *args, **kwargs):
        fields = kwargs.pop("fields", None)
        exclude = kwargs.pop("exclude", None)
        nest = kwargs.pop("nest", None)

        if nest is not None:
            if nest is True:
                self.Meta.depth = 1
            else:
                self.Meta.depth = None
                
        else:
            self.Meta.depth = None

        super(DynamicModelSerializer, self).__init__(*args, **kwargs)

        if fields is not None:
            # Drop any fields that are not specified in the `fields` argument.
            allowed = set(fields)
            existing = set(self.fields.keys())
            for field_name in existing - allowed:
                self.fields.pop(field_name)

        if exclude is not None:
            for field_name in exclude:
                self.fields.pop(field_name)
```
Now we are going to pass this serializer into one of our Model Serializers:

```py
class EventSerializer(DynamicModelSerializer):
    """serializer for events
    """
    class Meta:
      model = Event
      fields = ('id', 'game', 'description', 'date', 'time', 'organizer', 'joined')
```

This will allow us to modify the serializer dynamically to give us exactly what we want to return to our client.

## Using The Serializer

We are going to use query parameters from the request to dynamically modify our response. This allows the front end developers to have some control and autonomy over the data that they need. First we will need to add these query parameters to our list method:

```py
exclude = request.query_params.getlist('exclude')
fields = request.query_params.getlist('fields')
nest = request.query_params.get('nest')
```
the `getlist()` method returns a list of . That allows us to pass in multiple fields that we want to return or exclude multiple fields from the response.

Next we need to handle those query parameters. We are going to create a dictionary that we can pass to the serializer as keyword arguments, that way if nothing is passed the serializer will return the default response.

```py
serializer_params = dict()

if len(exclude):
    serializer_params['exclude'] = exclude

if len(fields):
    serializer_params['fields'] = fields

if nest is not None:
    serializer_params['nest'] = True
```

Finally, we need to pass this dictionary into our serializer:

```py
serializer = EventSerializer(events, many=True, **serializer_params)
```
## Example URLs



`http://localhost:8000/events?exclude=id`

This url will return the event objects as normal, but without the id.

`{
        "game": 8,
        "description": "Bongo Java Sorry Tournament",
        "date": "2022-12-23",
        "time": "19:00:00",
        "organizer": 2,
        "joined": true
    }`

`http://localhost:8000/events?fields=id&fields=game&nest=True`

This url will return the id and game attributes of the event with the game info nested.

`{
        "id": 7,
        "game": {
            "id": 8,
            "title": "Monopoly",
            "maker": "Parker Brothers",
            "number_of_players": 4,
            "skill_level": 2,
            "game_type": 1,
            "gamer": 2
        }
    }`
