---
layout: post
title: Django file generation and uploading to s3
---

Hello, In this post I will describe how to uplaod a file generated in django to AWS S3 and get a presigned url 
for downloading it. 



```
def upload_and_get_url(file_path, time=600, key=None):
    """ upload a file to s3 and get signed url """
    ext = file_path.split('.')[-1]
    key = key or (str(uuid4()) + '.' + ext)
    key = settings.S3_GENERATED_PATH + key
    transfer = S3Transfer(boto3.client('s3', settings.AWS_S3_REGION_NAME,
                                       aws_access_key_id=settings.AWS_ACCESS_KEY_ID,
                                       aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY))
    client = boto3.client('s3')
    transfer.upload_file(file_path, settings.S3_PRIVATE_BUCKET, key)
    url = client.generate_presigned_url(ClientMethod='get_object', Params={'Bucket': settings.S3_PRIVATE_BUCKET,
                                                                           'Key': key}, expires=time)
    return url

```

```
@task
def generate_file(data, filename, file_type, notify_user=None):
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
            df.to_csv(filename)
        elif file_type == 'excel':
            filename = temp.name + '.xls'
            df.to_excel(filename)
        url = upload_and_get_url(filename)
        temp.close()

    if notify_user:
        user = usr_mdl.User.objects.get(email=notify_user)
        notify.send(user, recipient=user, verb=f"{filename} file was successfully generated",
                    data={'type': 'file', 'details': str(url)})

    return url
```

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

    if num_records > settings.ASYNC_EXPORT_THRESHOLD:
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


```
    @action(detail=False, methods=['get'], url_path='poll-task')
    def poll_for_download(self, request):
        task_id = request.GET.get("task_id")

        if request.is_ajax():
            result = AsyncResult(task_id)
            if result.ready():
                return Response({"url": result.get()})
            return Response({"url": None})

        return Response({'result': 'file generation in progress'})

```
