# tcpprox


A simple TCP proxy written in GO. Allows for proxy-ing TCP connections as well as TLS wrapped TCP connections.

Can be run simply from the command-line using arguments. Or supplying a config file. Or both.
_The command line arguments have precidence and override the config file_

# Usage

To create a TLS proxy using the supplied config file:

```
tcpprox -s -c config.json -r 172.16.0.12:4550
```

To create a normal TCP proxy,  no config file:

```
tcpprox -l 0.0.0.0 -p 8081 -r 172.16.0.12:8081
```

To specify a custom certificate to use (PEM format) you can use the -cert and -key options (must be used together):

___Note (breaking change)__ for previous versions of tcpprox, the `-cert` and `-key` arguments were combined into one argument `-cert`. This previous arg would take the supplied value and automatically append **.pem** and **.key**. This is no longer the case and the supplied filepaths for `-cert` and `-key` must be complete and for valid, matching files._   

```
tcpprox -s -c config.json -cert server.pem -key server.key
```

To generate valid certificate and key:

```
openssl genrsa -out server.key 2048
openssl req -new -x509 -key server.key -out server.pem -days 3650
```

To convert the certificate to DER format:

```
openssl x509 -in server.pem -out server.crt -outform der
```


## mTLS

If proxying a connection where the upstream server uses mTLS, tcpprox can be configured to use mTLS to authenticate.

```
tcpprox -s -c config.json -cert server.pem -key server.key -clientCert client.pem -clientKey client.key
```

The application that is being proxied through tcpprox does not have to supply a client certificate (although it can). 

This allows for either:

```
client <---TLS---> tcpprox <---mTLS---> server
```

or

```
client <---mTLS---> tcpprox <---mTLS---> server
```

Tcpprox will allow both types of connections through, as long as tcpprox is able to use mTLS to connect to the server, the client is oblivious of what is happening upstream.

## Config File

The config file can be used instead of supplying all information on the command line. The options specified in the file will be overwritten by any matching command line arguments. This allows for using a config file and overriding one or more options for testing / variation between hosts.

# Using Docker

_note_: ensure you are running 'real' Docker [docker-ce](https://docs.docker.com/install/#supported-platforms)

**To build locally with Docker:**

```
docker build . -t staaldraad/tcpprox:latest
```

Or, even better, just get it directly off of Docker Hub

**Get from Docker Hub**

```
docker pull staaldraad/tcpprox:latest
```

**Run the container:**

```
docker run -it --rm -p 8000:8000 staaldraad/tcpprox:latest -p 8000 -s -r google.com:443
```

This will create a TLS enabled listener on port 8000 and proxy traffic to google.com:443


**Something about performance**

Method of outputting (displaying) the data being proxied determines how close to "non-proxied" performance you get.

* Running with `-raw -o /path/to/file` gives near native speed. AKA, it is as if the proxy isn't there. 
* Running with `-o /path/to/file` is almost as fast
* Outputting the hex.dump directly to screen is the slowest option. To get better speed, output hex to a file and use `tail -f /path/to/file`. It is still a little slower (depending on hardware) but better than direct to screen.

Direct to screen with hex dump formatting (default): `./tcproxy -p 8000 user@remote`
```
$ scp -P 8000 root@localhost:file.txt /tmp/p
file.txt                                                                                    100%   10MB   2.1MB/s   00:04    
```

Write to file with hex dump formatting: `./tcproxy -p 8000 user@remote -o /tmp/session`
```
$ scp -P 8000 root@localhost:file.txt /tmp/p
file.txt                                                                                    100%   10MB  10.2MB/s   00:00      
```

Write raw stream to file: `./tcproxy -p 8000 user@remote -o /tmp/session -raw`
```
$ scp -P 8000 root@localhost:file.txt /tmp/p
file.txt                                                                                    100%   10MB  10.7MB/s   00:00    

```

Write to file with hex dump formatting: `./tcproxy -p 8000 user@remote -o /tmp/session ` and use tail to view live stream: `tail -f /tmp/session`
```
$ scp -P 8000 root@localhost:file.txt /tmp/p
file.txt                                                                                    100%   10MB   5.9MB/s   00:01    

```

And raw (no output at all): `./tcproxy -p 8000 user@remote -raw`

```
$ scp -P 8000 root@localhost:file.txt /tmp/p 
file.txt                                                                                    100%   10MB  20.2MB/s   00:00     
```