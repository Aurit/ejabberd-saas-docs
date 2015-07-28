# XMPP support for HTTP file transfer

Author: Jérôme Sautret


The goal of this XMPP extension is to enable XMPP users to transfer
files using HTTP upload and download to a third-party HTTP file
hosting service.

The goal is to have the XMPP server facilitate the use of thirst party
file sharing tool while allowing the owner the file sharing account to
keep secret the "master key" that control upload and download
capability in the third-party file sharing service.

The protocol has been tested with Amazon S3. It should be compliant as
is with Riak CS server.

The plan is to test it with other providers to validate that the
workflow, while simple enough, cover the basic ideas of delegated file
sharing.

The protocol extension being used by ejabberd SaaS and not being an
official XEP uses custom namespaces for the moment.

## User Discovers Support

In order for a client to discover whether its server supports the
protocol defined herein, it MUST send a Service Discovery (XEP-0030)
information request to the server:

```xml
<iq to='server' type='get' id='disco1'>
  <query xmlns='http://jabber.org/protocol/disco#info'/>
</iq>
```

If the server supports the protocol defined herein, it MUST return a
feature of "p1:http-file-transfer":

```xml
<iq from='server' to='user@server/resource' type='result' id='disco1'>
  <query xmlns='http://jabber.org/protocol/disco#info'>
    ...
    <feature var='p1:http-file-transfer'/>
    ...
  </query>
</iq>
```

## File transfer use case

A client, connected with jid sender@domain.com/resource to the XMPP
server domain.com wants to send a file named "FILENAME" to the user
identified with jid "receiver@domain.com".

Using an intermediate HTTP storage, the file transfer can be
asynchronous and does not require both users to be online at the same
time like synchronous XMPP file transfer.

## File transfer XMPP extension

### Sender requests an upload URL

The client must first calculate the md5 sum of the file FILENAME, then
query an upload URL from the server using a set iq with the
base64-encoded 128-bit MD5 digest (according to RFC 1864) of the file
to upload as the md5 attribute of the query sub-element (of the iq
element) in the "p1:http-file-transfer" namespace:

```xml
<iq type='set' id='upload1' to='domain.com'>
  <query xmlns='p1:http-file-transfer' md5="qN48INKsmQyPMZkHs0DWog=="/>
</iq>
```

Note that the md5 digest is base64-encoded of the 128 bits binary one,
not its hexadecimal representation. You can verify that your md5
parameter is correct by comparing to the output of the following
command:

```shell
cat FILENAME | openssl dgst -md5 -binary | openssl enc -base64
```

### Server returns an upload URL and a file ID

The server returns an iq result with an URL and a file id:

```xml
<iq type='result' id='upload1' to='sender@domain.com/resource'>
  <query xmlns='p1:http-file-transfer' upload='https://bucket.s3.amazonaws.com/6e84ff49-8156-4d59-8108-1069cd3e78ad?AWSAccessKeyId=AKIAIJYQAA3X7VF2C6YQ&amp;Expires=1422975296&amp;Signature=x6zMZnWUxgNwOamwkpWID4X2gsc%3D' fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'/>
</iq>
```

### Sender uploads the file

The sender client uploads the file with a HTTP PUT to the upload URL
(returned by the server in the previous step, in the the upload
attribute of the query sub-element (of the result iq element) in the
p1:http-file-transfer namespace), with a content-md5 HTTP header set with
the same value that was passed in the md5 attribute in the first step,
and without setting a content-type in the headers of the HTTP request.

The upload URL expires after a configurable amount of time.

Example curl upload command may looks like this:

```shell
curl -X PUT -H "Content-MD5: qN48INKsmQyPMZkHs0DWog==" -H "Date: `date -R -u`" -T file_to_upload 'https://bucket.s3.amazonaws.com/6e84ff49-8156-4d59-8108-1069cd3e78ad?AWSAccessKeyId=AKIAIJYQAA3X7VF2C6YQ&amp;Expires=1422975296&amp;Signature=x6zMZnWUxgNwOamwkpWID4X2gsc%3D'
```

### Sender notifies the receiver

When the upload is completed, the sender sends the file name and file
id (returned previously by the server in the the fileid attribute of
the query sub-element (of the result iq element) in the
p1:http-file-transfer namespace) to the recipient in a message stanza:

```xml
<message type='chat' id='url1' from='sender@domain.com/resource' to='receiver@domain.com'>
  <body>sender@domain.com is sending you a file FILENAME with id 6e84ff49-8156-4d59-8108-1069cd3e78ad</body>
  <x xmlns='p1:http-file-transfer' name="FILENAME" fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'/>
</message>
```

### Receiver is notified of file to download

The receiver client receives the message stanza (either online or via
the offline storage), containing the file name and file id of the file
that sender wants to send him.

```xml
<message type='chat' id='url1' from='sender@domain.com/resource' to='receiver@domain.com'>
  <body>sender@domain.com is sending you a file FILENAME with id 6e84ff49-8156-4d59-8108-1069cd3e78ad</body>
  <x xmlns='p1:http-file-transfer' name="FILENAME" fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'/>
</message>
```

### Receiver asks a download URL to the server

The receiver asks the server for a download URL using a get iq using
the file id (received in the message stanza in the previous step) in
the fileid attribute of the query sub-element (of the iq element) in
the p1:http-file-transfer namespace:

```xml
<iq type='get' id='download1' to='domain.com'>
  <query xmlns='p1:http-file-transfer' fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'/>
</iq>
```

### Server returns download URL

The server returns an iq result with an URL in a download attribute of
the query sub-element (of the iq element) in the p1:s3filetransfer
namespace:

```xml
<iq type='result' id='download1' to='receiver@domain.com/resource'>
  <query xmlns='p1:http-file-transfer' download='https://bucket.s3.amazonaws.com/6e84ff49-8156-4d59-8108-1069cd3e78ad?AWSAccessKeyId=AKIAIJYQAA3X7VF2C6YQ&amp;Expires=1422975296&amp;Signature=aynmxSLHof6sAPqPqPqvV8l5eNWyM%3D'/>
</iq>
```

### Receiver downloads the file

The receiver client does a HTTP GET on the URL returned by the server in the previous step to retrieve the file.

The download URL expires after a configurable amount of time.

Example curl download command may looks like this:

```shell
curl -X GET -H "Date: `date -R -u`" 'https://bucket.s3.amazonaws.com/6e84ff49-8156-4d59-8108-1069cd3e78ad?AWSAccessKeyId=AKIAIJYQAA3X7VF2C6YQ&amp;Expires=1422975296&amp;Signature=aynmxSLHof6sAPqPqPqvV8l5eNWyM%3D'
```

