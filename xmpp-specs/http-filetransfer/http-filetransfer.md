# XMPP support for HTTP file transfer

Author: Jérôme Sautret


This XMPP extension allows for XMPP users to transfer files using HTTP
upload and download to a third-party HTTP file hosting service.

The goal is to have the XMPP server facilitate the use of third-party
file sharing tool while allowing the owner the file sharing account to
keep secret the "master key" that control upload and download
capability in the third-party file sharing service.

The protocol (and existing implementation) has been tested with Amazon
S3. It should be compliant as is with Riak CS server.

The plan is to test it with other providers to validate that the
workflow, while simple enough, cover the basic ideas of delegated file
sharing.

The protocol extension being used by ejabberd SaaS and not being an
official XEP uses custom namespaces for the moment.

*Note for ejabberd SaaS customers: As the specification is being
discussed on XSF Standards list and changing based on feedback, it
does not exactly match the production implementation. Please refer to
official production ejabberd SaaS documentation if you need actual
client deployment today.*

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
feature referenced as "p1:http-file-transfer":

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

A client, connected with jid "sender@domain.com/resource" to the XMPP
server "domain.com" wants to send a file named "FILENAME" to the user
identified with jid "receiver@domain.com".

Using an intermediate HTTP storage, the file transfer can be
asynchronous and does not require both users to be online at the same
time like synchronous XMPP file transfer.

### Sender requests an upload URL

The sending client MUST first calculate the md5 sum of the file
FILENAME, then query an upload URL from the server using a set iq with
the base64-encoded 128-bit MD5 digest (according to RFC 1864) of the
file to upload as the md5 attribute of the query sub-element (of the
iq element) in the "p1:http-file-transfer" namespace:

```xml
<iq type='set' id='upload1' to='domain.com'>
  <query xmlns='p1:http-file-transfer'
         md5="qN48INKsmQyPMZkHs0DWog=="/>
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

The server returns an iq result which MAY have the following fields as
attributes of the query:

* `upload`: the URL that the sender must use to upload the file to
  transfer
* `fileid`: a non-guessable string used to uniquely identity the file
  to be transferred
* `download`: URL used by the receiver to retrieve the file. This URL
  MAY expire after a given time, depending on the server setting. It
  MAY still be possible to genereate a new download URL using fileid
  if provided.
* `headers`: contains one or several "headers" elements, each having a
  "name" and "value" attributes:

The result MUST include the "upload" attribute.
The result MUST include at least one of "fileid" or "download"
attribute. The result MAY include both "fileid" and "download"
attributes.

The presence of "fileid" and "download" attributes depends of the
server policy regarding file transfer. For example, a server MAY offer
permanent download URL and only need to provide a download URL. In
other situations, a server may want to enforce XMPP-based access
control and will only provide download URL to receiver upon request
using the `fileid`.

The headers element MAY not be present.

Example of a iq result with both fileid and download attributes:

```xml
<iq type='result' id='upload1' to='sender@domain.com/resource'>
  <query xmlns='p1:http-file-transfer'
         upload='https://files.example.com/upload/x6zMZnWUxgNwOamwkpWID4X2gsc'
         fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'/>
</iq>
```

Example of a iq result with only fileid attribute and headers sub-element:

```xml
<iq type='result' id='upload1' to='sender@domain.com/resource'>
  <query xmlns='p1:http-file-transfer'
         upload='https://files.example.com/upload/x6zMZnWUxgNwOamwkpWID4X2gsc'
         fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'>
    <headers>
      <header name="x-auth-custom" value="MjzIOP7vKMQzCnNSkl"/>
      <header name="Content-MD5" value="qN48INKsmQyPMZkHs0DWog=="/>
    </headers>
</iq>
```

### Sender uploads the file

The sender client uploads the file with a HTTP PUT to the upload URL (returned by the server in the previous step, in the the upload attribute of the query sub-element (of the result iq element) in the p1:http-file-transfer namespace).

If the "headers" element was present in the previous request, they MUST be included in the HTTP headers passed for this PUT request.

The content-type MUST NOT be present in the headers of the HTTP request if it wasn't present in the "headers" element returned in the iq result in previous section.

The upload URL expires after a configurable amount of time.

Example curl upload command may looks like this:

```shell
curl -X PUT -H "Content-MD5: qN48INKsmQyPMZkHs0DWog==" -H "x-auth-custom: MjzIOP7vKMQzCnNSkl"-H "Date: `date -R -u`" -T file_to_upload 'https://files.example.com/upload/x6zMZnWUxgNwOamwkpWID4X2gsc'
```

### Sender notifies the receiver

When the upload is completed, the sender sends the file name and file
id (returned previously by the server in the the fileid attribute of
the query sub-element (of the result iq element) in the
p1:http-file-transfer namespace) and/or the download URL to the
recipient in a message stanza, using a "x" element in the
'p1:http-file-transfer' namespace, directly under the "message"
element.

If the sender has received the download URL in the iq result returned
by the server, he MAY include in the message stanza, as a "download"
attribute of that "x" element.

If the sender has received the fileid in the iq result returned by the
server, he MAY include in the message stanza, as a "fileid" attribute
of that "x" element.

The sender client MUST include at least either the download or the
fileid attribute into the message stanza.

The sender client SHOULD provide the info included into the "x"
sub-element in a plain text message in the "body" of the message, for
compatibility with a receiving client not supporting this protocol.

Example of stanza containing only the fileid:

```xml
<message type='chat' id='url1' from='sender@domain.com/resource' to='receiver@domain.com'>
  <body>sender@domain.com is sending you a file FILENAME with id 6e84ff49-8156-4d59-8108-1069cd3e78ad</body>
  <x xmlns='p1:http-file-transfer' name="FILENAME" fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'/>
</message>
```

Example of stanza containing the both the fileid and the download URL:

```xml
<message type='chat' id='url1' from='sender@domain.com/resource' to='receiver@domain.com'>
  <body>sender@domain.com is sending you a file FILENAME available at
  URL http://files.example.com/download/58351C33-9F3A-4D84-913C-BD6A2D8CED12/FILENAME</body>
  <x xmlns='p1:http-file-transfer' name="FILENAME"
     fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'
     download="http://files.example.com/download/58351C33-9F3A-4D84-913C-BD6A2D8CED12/FILENAME"/>
</message>
```

### Receiver is notified of file to download

The receiver client receives the message stanza (either online or via
the offline storage), containing the file name and info about the file
the sender wants to send:

```xml
<message type='chat'
         id='url1'
         from='sender@domain.com/resource' to='receiver@domain.com'>
  <body>sender@domain.com is sending you a file FILENAME available at
  URL http://files.example.com/download/58351C33-9F3A-4D84-913C-BD6A2D8CED12/FILENAME</body>
  <x xmlns='p1:http-file-transfer' name="FILENAME"
     fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'
     download="http://files.example.com/download/58351C33-9F3A-4D84-913C-BD6A2D8CED12/FILENAME"/>
</message>
```

If that message stanza contains the download URL, the receiver client
SHOULD perform a HTTP GET on this URL to retrieve the file: see
"Receiver downloads the file" below.

If the message stanza only contains the fileid, then the server MUST
ask the download URL to the server, see next section.

### Receiver asks a download URL to the server

The receiver asks the server for a download URL using a get iq using
the file id (received in the message stanza in the previous step) in
the fileid attribute of the query sub-element (of the iq element) in
the p1:http-file-transfer namespace:

```xml
<iq type='get' id='download1' to='domain.com'>
  <query xmlns='p1:http-file-transfer'
         fileid='6e84ff49-8156-4d59-8108-1069cd3e78ad'/>
</iq>
```

### Server returns download URL

The server returns an iq result with an URL in a download attribute of
the query sub-element (of the iq element) in the p1:http-file-transfer
namespace:

```xml
<iq type='result' id='download1' to='receiver@domain.com/resource'>
  <query xmlns='p1:http-file-transfer'
         download='http://files.example.com/download/58351C33-9F3A-4D84-913C-BD6A2D8CED12/FILENAME'/>
</iq>
```

### Receiver downloads the file

The receiver client does a HTTP GET on the URL returned by the server
in the previous step, or send directly by the sender, to retrieve the
file.

The download MAY URL expire after a configurable amount of time.

Example curl download command may looks like this:

```shell
curl -X GET -H "Date: `date -R -u`" 'http://files.example.com/download/58351C33-9F3A-4D84-913C-BD6A2D8CED12/FILENAME'
```

If the HTTP request responds that the URL has expired (using a for
example a 410 return code), AND the fileid was send in the message
stanza by the sender, then the receiver SHOULD ask a new URL to the
server: see "Receiver asks a download URL to the server" above.

## File sharing use case

If the sender wants to share a file with many peers, for a avatar
image for example, and if the server is configured to return a
download URL that doesn't expire, then the sender just have to perform
the "File transfer use case" steps up to "Sender uploads the file" and
then publish by any mean the download URL of the file, for example
following
[XEP-0084: User Avatar](http://xmpp.org/extensions/xep-0084.html)

## TODO

We plan to improve workflow to use ability to let the server file
transfer component requires other file hash types than binary-md5.
For example, the service could require SHA-1 hash instead, just file
size check or no hash.

It is likely that this case will be resolved by an XMPP data form.

We are waiting for your feedback :)
