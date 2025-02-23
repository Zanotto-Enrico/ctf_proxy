# ctf_proxy - A TCP proxy for intercepting and dropping malicious attacks

This tool is purposely made for Attack/Defence CTF competitions. 
A proxy process is created for each configured service, with the possibility to block malicious attacks by means of custom filters.

## Configuration
You can configure each service to be proxied using `proxy/config/config.json`.

```json
{
    "services": [
        {
            "name": "generic_service", 
            "target_ip": "10.10.10.10", 
            "target_port": 80,
            "listen_port": 80
        },
        {
            "name": "ssl_service",
            "target_ip": "localhost",
            "target_port": 443,
            "listen_port": 500,
            "http": true,
            "listen_ip": "0.0.0.0",
            "ssl": {
                "server_certificate": "server.pem",
                "server_key": "server.pem",
                "client_certificate": "client_crt.pem",
                "client_key": "client_key.pem"
            }
        }
    ],

    "global_config": {
        "keyword": "KEYWORD FOR PACKET ANALYZERS",
        "verbose": false,
        "dos": {
            "enabled": true,
            "duration": 60,
            "interval": 2
        },
        "max_stored_messages": 10,
        "max_message_size": 65535
    }
}
```
### Parameters
In the `services` list, the following parameters can be set for each service:

- **name**: name that will be used for logging and filter modules generation
- **target_ip**: service IP/hostname
- **target_port**: service port
- **listen_port**: proxy port to listen on
- **listen_ip**: *(optional)* IP where the proxy will listen on, default=```"0.0.0.0"```
- **http**: *(optional)* must be set to `true` to enable HTTP parsing, default=```False```
- **ssl**: *(optional, required for SSL services)* set SSL files for secure communication. Each file will be looked in the `proxy/config/certificates` folder:
  - **server_certificate**: server certificate in PEM format
  - **server_key**: server key file
  - **client_certificate**: *(optional)*: for client authentication, client certificate in PEM format
  - **client_key**: *(optional)*: for client authentication, client key file

The `global_config` contains:
- **keyword**: string to be sent as response to malicious packets, to facilitate packet inspections
- **verbose**: verbose mode
- **dos**: slow down attackers by keeping the socket alive:
    - **enabled**: *(boolean)*
    - **duration**: *(seconds)* time to wait before closing the socket
    - **interval**: *(seconds)* time interval between packets to keep the socket alive
- **max_stored_messages**: maximum number of stored messages in the `previous_messages` queue in the Stream objects
- **max_message_size**: maximum message size after which the message will be truncated before being stored in the `previous_messages` queue in the Stream objects

## Usage
The tool can be used as a Docker container or as a CLI Python application.

### Docker Example
Clone the repository:
```
git clone https://github.com/ByteLeMani/ctf_proxy
cd ctf_proxy
```
Edit the docker-compose.yml file with the current port mapping. For example, if you want to listen on port 8080 and 9090, you need to set:
```yml
ports:
  - 8080:8080
  - 9090:9090
```

Then run the container:
```bash
docker compose up --build -d
```
#### FOR A/D CONTAINERIZED SERVICES
You may want to add this configuration to the services docker-compose.yml file and ***remove any address/port binding*** it may have, to take advantage of the docker network DNS.
```yml
networks:
  default:
    name: ctf_network
    external: true
```
This way, you can use the services' hostname directly in the `target_ip` parameters. Moreover, as the services are connected to the proxy network they are reachable inside it without exposing or changing any port, but not reachable from the outside.

### Automatic script
All the above configuration can be done with the provided setup script. Either clone the repository, install ruamel.yaml and run the script or directly run
```
pip install ruamel.yaml; curl -s https://raw.githubusercontent.com/ByteLeMani/ctf_proxy/main/setup_proxy.py > setup_proxy.py; python3 setup_proxy.py
```
The script will work as long as it is called from inside ctf_proxy or besides all the services. In the latter case it will clone ctf_proxy before configuring it.
You can manually provide paths for the services as arguments when you call the script. If you don't it will automatically scan cwd and look for services, asking for confirmation.

### CLI Example
Clone the repository, install the required packages and run it:
```bash
git clone https://github.com/ByteLeMani/ctf_proxy
cd ./ctf_proxy/proxy
pip install -r requirements.txt
python3 proxy.py
```

## Modules
These are the core filtering entities of the proxy. For each proxied service, a pair of modules is ***automatically generated*** inside the `proxy/filter_modules/<service_name>/` folder the first time you run the proxy. Modules execution will follow this flow: 

![proxy](https://user-images.githubusercontent.com/93737876/222983045-c3a8237a-4b43-40e4-9dcb-302fd3642362.jpg)

Inside the modules you will find an `execute` method that receives a ***Stream object*** from the proxy and returns to the proxy whether the stream contains an attack or not. If an attack is found, the proxy will send to the attacker a custom string (```KEYWORD + " " + SERVICE NAME + " " + ATTACK NAME```) to easily find attacks in PCAP files if a packet analyzer is used in the system. At this point, the socket will be kept alive to interfere with attackers' scripts if the `dos` parameter is enabled, otherwise the socket will be closed.

:warning: Note that `dos` may not work properly if `ssl` is enabled :warning:

### Stream
**Stream** is the interface for **TCPStream** and **HTTPStream** and contains the current and previous received messages on the socket.

#### TCPStream
TCPStream is used in case of TCP only connections. The `previous_messages` queue contains the latest `max_stored_messages` messages that went through the socket before the current received message. The latter will be stored in the `current_message` variable and can be modified to alter the content of the message that will be sent from the proxy.
#### HTTPStream
HTTPStream is used in case of HTTP connections. Now two more variables are available: `previous_http_messages` and `current_http_message`. They are the parsed version of the correspective raw variables as `HttpMessage` objects. The actual values sent through the socket are the ones stored in `current_message`, which means you should edit this variable to alter the content of the message sent from the proxy.

:warning: Watch out for `Content-Length` header in the raw HTTP message if you intend to modify it. :warning:

### Update module
To add a new filter, define a new function inside the class Module called as the name of the attack (or a custom one if you prefer) that accepts a Stream object as parameter and returns a boolean (True if attack found, False if not). You will find a filter example in `proxy/filter_modules/template.py`.

Every module will be ***automatically reloaded on the fly*** by simply modifying it. If an exception is thrown during the import, the module will not be loaded and the previous version will be used instead. If an exception is thrown at runtime, the packet will simply flow through the proxy.

### Database
To add persistance to the filters, you can build and use the local Mongo database. You can access the database inside the modules through the `DBManager` interface. You can find some examples in `proxy/filter_modules/example_functions.py`.

## Logging
A simple log file `proxy/log.txt` will count all the blocked packets for each service. If the file already exists at startup, the proxy will update the counts based on their initial value inside the file. Otherwise, the file will be created at startup.

You can monitor the file by using:
```
watch cat log.txt
```
Any relevant information will be printed to stdout. Set `verbose` to `true` for more info.

If you're using the Docker version, you can inspect the docker logs to access them: 
```
docker logs --follow proxy
```
## Credits
Written by:
- [fcirillo00](https://github.com/fcirillo00)
- [AleDecre](https://github.com/AleDecre)
- [oruamdalo](https://github.com/oruamdalo)

This tool is inspired by and partially based on the TCP proxy logic of [tcpproxy.py](https://github.com/ickerwx/tcpproxy/blob/master/tcpproxy.py).
