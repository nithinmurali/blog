### How to upload both file and json using django rest framework in same api endpoint

After spending 1 day on this, I figured out that ...

For someone who needs to upload a file and send some data, there is no straight fwd way you can get it to work. There is an [open issue][1] in json api specs for this. One possibility i have seen is to use `multipart/related` as shown [here][2], but i think its very hard to implement it in drf.

Finally what i had implemented was to send the request as `formdata`. You would send each file as _file_ and all other data as text.
Now for sending the data as text you have two choices. case 1) you can send each data as key value pair or case 2) you can have a single key called _data_ and send the whole json as string in value. 

The first method would work out of the box if you have simple fields, but will be a issue if you have nested serializes. The multipart parser wont be able to parse the nested fields.

Below i am providing the implementation for both the cases

Models.py
```
    class Posts(models.Model):
        id = models.UUIDField(default=uuid.uuid4, primary_key=True, editable=False)
        caption = models.TextField(max_length=1000)
        media = models.ImageField(blank=True, default="", upload_to="posts/")
        tags = models.ManyToManyField('Tags', related_name='posts')
```

serializers.py -> no special changes needed, not showing my serializer here as its too lengthy because of the writable ManyToMany Field implimentation.

views.py
```
    class PostsViewset(viewsets.ModelViewSet):
        serializer_class = PostsSerializer
        #parser_classes = (MultipartJsonParser, parsers.JSONParser) use this if you have simple key value pair as data with no nested serializers
        #parser_classes = (parsers.MultipartParser, parsers.JSONParser) use this if you want to parse json in the key value pair data sent
        queryset = Posts.objects.all()
        lookup_field = 'id'
```
Now, if you are following the first method and is only sending non-Json data as key value pairs, you don't need a custom parser class. DRF'd MultipartParser will do the job. But for the second case or if you have nested serializers (like i have shown) you will need custom parser as shown below.

utils.py
```
    from django.http import QueryDict
    import json
    from rest_framework import parsers
    
    class MultipartJsonParser(parsers.MultiPartParser):
    
        def parse(self, stream, media_type=None, parser_context=None):
            result = super().parse(
                stream,
                media_type=media_type,
                parser_context=parser_context
            )
            data = {}

            # for case1 with nested serializers
            # parse each field with json
            for key, value in result.data.items():
                if type(value) != str:
                    data[key] = value
                    continue
                if '{' in value or "[" in value:
                    try:
                        data[key] = json.loads(value)
                    except ValueError:
                        data[key] = value
                else:
                    data[key] = value

            # for case 2
            # find the data field and parse it
            data = json.loads(result.data["data"])

            qdict = QueryDict('', mutable=True)
            qdict.update(data)
            return parsers.DataAndFiles(qdict, result.files)
```
This serializer would basically parse any json content in the values.

The request example in post man for both cases: 

Case 1

[![case 1][3]][3], 

Case 2

[![case2][4]][4]


  [1]: https://github.com/json-api/json-api/issues/246
  [2]: https://cloud.google.com/storage/docs/json_api/v1/how-tos/multipart-upload
  [3]: https://i.stack.imgur.com/xgYod.png
  [4]: https://i.stack.imgur.com/2hokM.png
