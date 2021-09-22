# Protect HLS streaming with CloudFront signed URL and Lambda@Edge

This code sample shows how you can protect HLS streaming, more precisely all of the urls in the streaming with signed URL, when you can't rely on signed cookie.  

## HLS streaming  
HLS streaming consists of playlist files (`*.m3u8`, also called as manifest files) and segment files (`*.ts`).  
Playlist file contains list of other playlist file urls or segment file urls, and streaming player will continuously download them and play the media.  

## Protecting HLS streaming with CloudFront signed URL
CloudFront provides a feature called as signed URL / signed cookie that will verify whether the incoming HTTP request has correct policy and signature.  
Only the requests with right policy and its signature that is digitally signed with private key will get the content from CloudFront, and other request will be denied.  
You can protect your HLS streaming by using this signed URL or signed cookie and give signed values only to authenticated users.  
Typical look of the signed URL will be like below:
```
https://d12345678.cloudfront.net/stream/index.m3u8?Signature=ot71ZXg2-m6xOWGrIQIC4Euggj2rlb4ONfIhe5VZ7zUUZjNPMXNiUuWDhdNf6P97i0v2BuF6MgBF3dywJ-vbPOTS9Nc-L1PXTACdjAQMcwn7NJ3df1B~ropRJCrcH0WvuslcphLjNzBiSFj1R61rklyrSaa0hdZtu8v0to~ukKJwl8kl61vIrCZCsELo9VkByy9KByt7pudWWuHUqEufgmermsUOcmD9RHaluRw-0D3tMMAAG0asS24J6lFn7L3SQekkZH1fRO8DEQX49GvsWf1OGxcF-X39bsOKzm3CzVyCjJjlonOwNKeUdq8xGd9B8~YA5ftPYvOeUIKxuPyinA__&Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9kMmM1OXJza3BhMGo3di5jbG91ZGZyb250Lm5ldC9vdXQvdjEvM2ZiMTA0NTI3OGRhNDRjMWE4MjU3OTIyNjdkMGFhNmMvZGU0YjFjM2MxMDlhNGEyM2I1MjQzYjM3ZDUwYmJhMjIvN2MzODM3NDZhNDZmNGQ2MWE3NjU3MjU1M2NkYjZlZWUvaW5kZXgubTN1OCIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTYzMTcwMDAwMH19fV19&Key-Pair-Id=K4EX1DBNJN55U
```

## Updating HLS streaming playlist file
When protecting HLS streaming with CloudFront signed value, signed cookie is desired since all of the requests will automatically have the signed values, and client doesn't need to add any extra query string.  
However, sometimes using cookie is not available option for particular case, which makes it difficult to protect whole segment urls.  
Below is a simple example of a playlist file:  
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:9
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:8.333,
stream/index_1_0.ts
#EXTINF:8.333,
stream/index_1_1.ts
#EXTINF:8.333,
stream/index_1_2.ts
#EXTINF:8.833,
stream/index_1_3.ts
```
When the streaming can't use the streaming, each urls in the playlist file should have the signed values:
```
#EXTINF:8.333,
stream/index_1_0.ts?Signature=ot71ZXg2-m6x...&Policy=eyJTdGF0Z...&Key-Pair-Id=K4EX...
#EXTINF:8.333,
stream/index_1_1.ts?Signature=ot71ZXg2-m6x...&Policy=eyJTdGF0Z...&Key-Pair-Id=K4EX...
```
So to protect HLS stream, you need to update(rewrite) playlist file to contain those signed values on the fly.  

## How to use this code

**Prerequisite:** You need to have your streaming origin, and bash shell.

1. Download the CloudFormation template file (`protect_hls.yaml`)

1. Create a key pair to make the signed URL  
bash: `openssl genrsa -out CF-priv-key.pem 2048; openssl rsa -pubout -in CF-priv-key.pem -out CF-pub-key.pem;`  

1. Make the private key to one line with embedded newline character (\n)  
bash: `awk 'NF {sub(/\r/,""); printf "%s\\n", $0;}' CF-priv-key.pem`

1. Replace `<PASTE YOUR PRIVATE KEY HERE>` with the one lined private key value you generated. It will look something similar like below;

```
          - |
            ";
                let privateKey = "-----BEGIN RSA PRIVATE KEY-----\nMIIB....\nAAAA....\n-----END RSA PRIVATE KEY-----\n";
                let duration = 300;
```

1. Replace `<PASTE YOUR PUBLIC KEY HERE>` with the public key value. Use multi line as original file. it will look like below;

```
        EncodedKey: |
          -----BEGIN PUBLIC KEY-----
          MIIB....
          SpJr....
          AAko....
          ........
          -----END PUBLIC KEY-----
```

1. Go to AWS management console > US East (N. Virginia) region > CloudFormation
1. 'Create stack' with 'Upload a template file' option. Choose the `protect_hls.yaml` file as the template.
1. 

# Implementation

There are 2 CloudFront distributions, one is called as viewer distribution, and another is manifest distribution.  
Viewer distribution will deliver all of the streaming to end user, while manifest distribution is used only to cache manifest files.  
1. When a player opens a manifest, CloudFront will trigger Lambda@Edge function as origin request trigger.  
1. Lambda@Edge will read the manifest file, calculate signed URL for each URL in it, and update those URLs.  
1. This manifest will be served to player
![when reading manifest](reading_manifest.png)

1. When a player opens a segment file, it will use the signed URL
1. Viewer distribution will validate the signed value, and serve segment files
![when reading segments](reading_segment.png)