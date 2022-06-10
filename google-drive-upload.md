# Google Drive Upload Notes

## Why?

I need to write a small utility to backup something and I decide to use Google Drive as my remote storage. So I checked Google Drive API documents then I noticed that though the documents are pretty comprehensive, they don't explain something well and it's easy to misunderstand the usages. So I would like to write down my findings and show some codes to explain how it works.

And the funny thing is, I see many different methods to access Google Drive API if I search in StackOverflow and I didn't see any of them matches what Google Drive API documents describes.

This note only talks about Google Drive API v3 upload part and **DOES NOT** include Google Drive SDK usage. I will use HTTP REST APIs only to access Google Drive.

## Prerequisites

Okay, before moving on, the reader should have some basic knowledge to understand how OAuth2 authorization protocol works. I won't explain it here because that's another big topic. Google Drive API only permits OAuth2 to grant authorization so do some homework is very necessary, Here's a good explanation [document](https://developers.google.com/identity/protocols/oauth2).

Also, it is important to understand the *SCOPE* concept in Google Drive API. Here's the [document](https://developers.google.com/drive/api/v3/about-auth).

To use any of Google APIs, the user needs to create an application associated with *client ID* and *client secret*. Those are critical credentials to generate tokens for further access on Google services. Here are high-level steps:

1. [Create a Google App Project](https://developers.google.com/workspace/guides/create-project)
2. [Create credentials](https://developers.google.com/workspace/guides/create-credentials)
3. In *Credentials* page, change **Authorized redirect URIs** in GCP console to **https://developers.google.com/oauthplayground**
4. Save Client ID and Client secret for further use

This is the reason why I use [OAuth2 Playground](https://developers.google.com/oauthplayground/): I need to generate OAuth2 **refresh token** to help me generate **access token**. Once I have the refresh token I can place it in my code and start to generate access token. In summary, in order to generate the access token, I need client ID, client secret and refresh token. As we can see, using OAuth2 Playground is a one-time thing and just for acquiring the refresh token.

Once I have the access token, I can access Google Drive API.

## OAuth2 Playground Setup

Alright, I have my client ID and client secret, how to set up the environment to get the refresh token? Just click the gear icon on the top right and fill out following values:

| Name | Value |
| --- | --- |
| OAuth flow | Server-side |
| Access type | Offline |
| OAuth Client ID | client ID |
| OAuth Client secret | client secret |

Then select **Drive API v3** and choose https://www.googleapis.com/auth/drive scope then authorize Google OAuth2 Playground to access the Google account who created the Google Drive project.

Once the above step is done, click the **Exchange authorization code for tokens** button in Step 2 on the screen to generate the **refresh token**. The access token expires in 1 hour, I can refresh it by using refresh token that I just got. Now, I have everything I need to access Google Drive API.

## Talk is cheap, show me the code

When I start to explore the Google Drive API documents, I found there are several different pages to talk about uploading files but I could not combine them into a solution easily. Well, maybe I'm stupid :-D.

Anyway, let me list them first so readers can have a brief idea what they are.

[Create Files](https://developers.google.com/drive/api/v3/create-file)

[Upload File Data](https://developers.google.com/drive/api/v3/manage-uploads)

[Files: Create](https://developers.google.com/drive/api/v3/reference/files/create)

[Files: Update](https://developers.google.com/drive/api/v3/reference/files/update)

This is my suggestion: please read above documents multiple times to understand how to upload a file in a proper way by using Google Drive API. I took a good amount of time to figure out the correct path.

So, if readers want to skip the dizzy part, here is the conclusion: to upload a file in Google Drive, the file should be created first in the drive with proper metadata then uploading the file content data to the file.

Wait a second, does it mean there must be 2-step to upload a file in Google Drive? Of course not! But let me make this loud and clear: this 2-step way is *the* proper way if [Simple Upload](https://developers.google.com/drive/api/v3/manage-uploads#simple) is the option and I will show the readers how to do that in Python. I use [requests](https://docs.python-requests.org/en/master/) module to handle HTTP requests and responses.

I will explain the whole steps in 3 parts: **Authentication**, **Create a new file** and **Upload file content data**.

### Authentication

To access Google Drive API, I need client ID, client secret and access token. And the access token is generated & refreshed by client ID, client secret and refresh token.

HTTP **POST** method is required to get the access token. Here's a simple example:

```python
oauth2_post_body = {'client_secret': client_secret, 'client_id': client_id, 'grant_type': 'refresh_token', 'refresh_token': refresh_token}

oauth2_client = requests.session()
oauth2_response = oauth2_client.post('https://oauth2.googleapis.com/token', data = oauth2_post_body)

if oauth2_response.status_code == 200:
    oauth2_access_token_response = oauth2_response.json()
    access_token = oauth2_access_token_response['access_token']
else:
    pass

oauth2_response.close()
```

Above codes can be wrapped as a function to get & refresh the access token.

### Create a new file

This step is to create an empty file with necessary metadata in Google Drive. As I only use Simple Upload so I don't need to pass lots of metadata. However, IMHO, MIME Type is a critical one in the request and must be defined properly. To view all property names in POST request body, please read [Files: Create](https://developers.google.com/drive/api/v3/reference/files/create).

HTTP **POST** method is required to create a new file. Here's a simple example:

```python
create_file_post_headers = {'Authorization': 'Bearer '+access_token, 'Content-type': 'application/json'}
create_file_post_body = {'name': filename, 'parents': [folder_id], 'mimeType': 'application/x-tar'}

oauth2_client = requests.session()
oauth2_response = oauth2_client.post('https://www.googleapis.com/drive/v3/files', headers = create_file_post_headers, data = json.dumps(create_file_post_body))

if oauth2_response.status_code == 200:
    create_file_response = oauth2_response.json()
    file_id = create_file_response['id']
else:
    pass
    
oauth2_response.close()
```

Once a new file is created successfully, a new file ID will be assigned to this remote file. Users need to keep it for the file content data uploading.

### Upload file content data

To upload the file content, the file data needs to be read into a variable then pass to a HTTP **PATCH** method. To understand HTTP PATCH method, here's the [document](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/PATCH) from Mozilla MDN.

Remember to use binary mode to read the file content. Here's a simple example:

```python
with open(local_filename, 'rb') as f:
    file_data = f.read()

update_file_post_headers = {'Authorization': 'Bearer '+access_token, 'Content-type': 'application/x-tar'}

oauth2_client = requests.session()
oauth2_response = oauth2_client.patch('https://www.googleapis.com/upload/drive/v3/files/'+file_id+'?uploadType=media', headers = update_file_post_headers, data = file_data)

if oauth2_response.status_code == 200:
    update_file_response = oauth2_response.json()
    update_file_id = update_file_response['id']
else:
    pass

oauth2_response.close()
```

Once the file content data is uploaded successfully, Google Drive API will respond a JSON block with the remote file ID that I created above.

## End

Well, that's it. I don't cover too much in this note. Hope this helps anyway.

I know some users would like to use [curl](https://curl.se/) utility to upload the file content. It's absolutely doable but there's a small trap when passing `Content-Type`. From curl manual:

>       --data-binary <data>
>              (HTTP) This posts data exactly as specified with no extra processing whatsoever.
>
>              If  you  start  the  data with the letter @, the rest should be a filename.  Data is posted in a similar >manner as -d,
>              --data does, except that newlines and carriage returns are preserved and conversions are never done.
>
>              Like -d, --data the default content-type sent to the server is application/x-www-form-urlencoded. If you >want the data
>              to  be treated as arbitrary binary data by the server then set the content-type to octet-stream: -H >"Content-Type: ap‐
>              plication/octet-stream".

Which means, if I've defined MIME Type when creating the file in Google Drive, I could only use `Content-Type: application/octet-stream` in curl to make sure it does not mess up the MIME Type of the file in Google Drive.

Comments and suggestions are welcome. Thanks.
