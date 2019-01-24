reduce statements in dockerfile to reduce image size

you need to run collectstatic while building image, hence set a dummy secrect_key while docker building and for all other options make sure 
to set a default value in settings

before runing container, migrations needs to be present in the image
