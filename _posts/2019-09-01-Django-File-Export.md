---
layout: post
title: File export in Django using S3
---

Hi, 

There are 3 ways to export files in django

* export from django request
* export using an async task (celery)
* export using an async task but upload the generated file to s3 and provide its link

In this post we will see how we can use a combination of 1 and 3, to dynamicaly generate and upload files.
So out final application will take some data and will geenerate a file based the requested format, will upload it to a priate 
S3 bucket. Then will give link for the users to download the file. The link will expire after x seconds and so does the file.

First create a private bucket in S3 and create a folder inside it with the name 'generated', add  a lifecycle rule for all 
items in the generated folder to expire aftre 1 day.

Then configure your aws credentials in django settings, I use python decouple, hence the `config`. But you can just write the key there.

```
  AWS_ACCESS_KEY_ID = config('AWS_ACCESS_KEY_ID')
  AWS_SECRET_ACCESS_KEY = config('AWS_SECRET_ACCESS_KEY')

  S3_PRIVATE_BUCKET = 'nymbleup-private'
  S3_GENERATED_PATH = 'generated/' # path where generated files are stored in private bucket
  ASYNC_EXPORT_THRESHOLD = 100 # number of rows after which file generation should be pushed to queue
```
Below is the function to upload a file to amazon s3 and generate a presigned url for that file

```
def upload_and_get_url(file_path, time=600, key=None):
    """ upload a file to s3 and get signed url 
    
    :param file_path: path of the file to upload
    :param time: expiration time of the generated url
    :param key: name of the file when stored in s3    
    """
    ext = file_path.split('.')[-1]
    key = key or (str(uuid4()) + '.' + ext)
    key = settings.S3_GENERATED_PATH + key
    transfer = S3Transfer(boto3.client('s3', settings.AWS_S3_REGION_NAME,
                                       aws_access_key_id=settings.AWS_ACCESS_KEY_ID,
                                       aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY))
    client = boto3.client('s3')
    transfer.upload_file(file_path, settings.S3_PRIVATE_BUCKET, key)
    url = client.generate_presigned_url('get_object', Params={'Bucket': settings.S3_PRIVATE_BUCKET,
                                                                           'Key': key}, ExpiresIn=time)
    return url

```

Now lets generate the file and call the above functoin.

```

@task
def generate_file(data, filename, file_type):
    """
     generate a file from data , upload to s3 and get url

    :param data: list of dicts
    :param filename: string
    :param file_type: csv or excel
    """
    if not data:
        return ""

    df = pd.DataFrame(data)

    with tempfile.NamedTemporaryFile() as temp:
        if file_type == 'csv':
            filename = temp.name + '.csv'
            df.to_csv(filename, index=False)
        elif file_type == 'excel':
            filename = temp.name + '.xls'
            df.to_excel(filename, index=False)
        else:
            raise BadInputError('invalid file format')
        url = upload_and_get_url(filename)
        temp.close()

    return url

```

Note that the above function is a celery task, and will be run in async mode.

```

def export_file_response(data, file_name, file_type, queryset=None, user=None):
    """
    get the response for exporting file, will switch to async if file is longer

    :param data: data, list of dicts
    :param file_name:
    :param file_type: csv or excel
    :param queryset: if no data is provided take data from queryset
    :returns: will return task id which can be used to poll progress
    """
    num_records = len(data)
    if not data:
        data = queryset
        num_records = queryset.count()

    if num_records > settings.ASYNC_EXPORT_THRESHOLD and file_type not in ['csv']:
        task = generate_file.delay(data, file_name, file_type, user)
        return Response({"task_id": task.task_id})

    if file_type == 'csv':
        response = Response(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="users.csv"'
        if not data:
            return responses.response_ok("no data to export")
        keys = data[0].keys()
        dict_writer = csv.DictWriter(response, keys)
        dict_writer.writeheader()
        dict_writer.writerows(data)
        return response
    else:
        raise BadInputError('invalid file type for large data')

```

This function will call the file generation code if the number of rows in data is greater than `ASYNC_EXPORT_THRESHOLD`.
else will generate a csv response and will return that. The file generation code is called in async mode, so the there should be some way to let the user know that generation is complete. For this either you can send an email or notification once the generation is complete, or can have the ui poll to know the status.

Here is the view code for polling.

```
    def poll_for_download(self, request):
        task_id = request.GET.get("task_id")

        if request.is_ajax():
            result = AsyncResult(task_id)
            if result.ready():
                return Response({"url": result.get()})
            return Response({"url": None})

        return Response({'result': 'file generation in progress'})

```


