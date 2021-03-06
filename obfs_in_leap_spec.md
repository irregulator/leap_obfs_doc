##obfsproxy in LEAP

##Overview

This is a documentation concerning the integration of obfsproxy traffic obfuscation framework in the LEAP ecosystem. It specifies how should the obfsproxy function and defines a threat model against which LEAP's VPN over obfsproxy would be effective. It contains a section about pluggable transports used by obfsproxy. Finally, it gives information on how obfsproxy should be treated both by the LEAP administrator and the LEAP end-user point of view.

##Specification

1. LEAP provider should be able to deploy obfsproxy as a service in any node, either alonside another service or standalone.
2. obfsproxy servers should be able to support multiple pluggable transports.
3. bitmask client should be able to find out provider's obfsproxy servers' connection details
4. bitmask client should be able to launch obfsproxy and route VPN traffic through the obfuscated channel.
5. A passive adversary anywhere between client and the provider's node should not be able to distinguish VPN traffic

###Threat Model

provider' side assumptions:

   * there is at least one obfsproxy server listening
   * obfsproxy server forwards traffic to a provider's VPN gateway

client's side assumptions:

   * client has a valid copy of bitmask
   * client operates within a network that blocks VPN traffic
   * client is able to login with a provider, download OpenVPN client certificates and 'eip-service.json'
   * client uses VPN over obfsproxy

adversary's side assumptions:

   * adversary is able to identify VPN traffic, e.g. by using Deep Packet Inspection technology
   * adversary blocks VPN traffic
   * adversary can opportunistically follow client's connections to probe and identify obfsproxy servers with legacy pluggable transports like 'obfs2', 'obfs3'.

Then:
    
   * adversary will not be able to distinguish VPN traffic between client and provider's obfsproxy node
   * adversary will not be able to probe and identify an obfsproxy server if scramblesuit is used as transport and adversary knows not the shared secret
   * adversary will still be able to see client's traffic destinations
   * adversary can still observe HTTPS traffic between client and the provider's API (register or log into account, get client certificates, service definition files)

##Pluggable Transports

Pluggable  transports are protocols that transform underlying network traffic in a  certain way. Pluggable transports are used by obfsproxy in Tor  ecosystem, so as to obfuscate Tor traffic between client and its first  hop (the bridge) in a Tor circuit.

obfsproxy  is meant to support arbitrary number of pluggable transports in a  modular way. This is needed since there is an arms race between  technology of censorship and technology resisting censorship. In other  words, new pluggable transports are introduced trying to mitigate  existing ways to identify and censor traffic, then censorship might  eventually adapt and be able to the new transports and so forth.

During  the last years a number of pluggable transports has been developed,  where each of them introduces a certain way to obfuscate traffic and has  different features.

Some  pluggable transports make the traffic look like random, e.g 'obfs3',  'scramblesuit'. 'FTE' transforms traffic according to a certain regular  expression/pattern, while 'BananaPhone' tries to resemble natural  language. 'Meek' uses TLS and HTTP and relays traffic through a third  party server that a censor will find too expensive to block, e.g.  Google's app engine.

###PT status in Tor ecosystem

As  of August 2014, bridges distributed by the BridgeDB of Torproject, may  support one ore more of four pluggable transports: 'obfs2', 'obfs3',  'scramblesuit' and 'FTE'. 

'obfs2'  and 'obfs3' although widely deployed are considered trivial to be  identified and blocked by censors. Specifically 'obfs2' handshake is  easily spotted by a passive adversary. 'obfs3' avoids that by using  Uniform Diffie-Hellman, but is susceptible to active probing by the  adversary, an attack actually observed in the wild.

Scramblesuit is a fresh pluggable transport which introduces some nice features in traffic obfuscation. Particularly:
mitigates active probing by using a shared secret
mitigates traffic flow analysis by modifying packets' length and inter-arrival time

Although  'scramblesuit' is already deployed in some Tor bridges, Tor project  decided to hold scramblesuit aside as an "emergency" pluggable transport  and instead try to deploy a newer one on Tor bridges. This transport,  temporarily called'obfs4', incorporates scramblesuit's features but with  some variations: uses ntor handshake obfuscated with Elligator instead  of UniformDH handshake, uses NaCl secretbox instead of AES-CTR for  application data, is implemented in Go instead of Python. 'obfs4' has  not yet been deployed in Tor bridges.

https://trac.torproject.org/projects/tor/wiki/doc/AChildsGardenOfPluggableTransports

###PT in LEAP

obfsproxy  integration in LEAP should support any pluggable transport supported by  the upstream. First approach of the integration described below was  done with 'scramblesuit' in mind. Other PT should also work with slight  modifications.

##Software additions and modifications

Following paragraphs reflect a first approach of integrating obfsproxy in LEAP ecosystem, as part of GSoC project, summer 2014.

###leap\_platform

A obfsproxy puppet module is available in leap\_platform. This module takes care of all the system requirements to have a obfsproxy service instance in a specified LEAP node.

obfsproxy puppet module will address the creation of system user and group, configuration directory and file, log file, package installation and service status.

Puppet class 'obfsproxy' needs the following arguments: 
    $transport, 
    $bind\_address, 
    $port, 
    $param, 
    $dest\_ip, 
    $dest\_port, 
    $log\_level = 'info'
    
These values are passed to '/etc/obfsproxy/obfsproxy.conf' via a erb template.

Puppet module will place a init script under /etc/init.d/ which will launch obfsproxy as daemon. Init script provides the basic "start-stop-status" functionality, so as the system can treat obfsproxy as service. The init script sources '/etc/obfsproxy/obfsproxy.conf' and launches a listener with parameters defined in the configuration file.

obfsproxy listener will bind to the predefined port ($port) and IP address ($bind\_address) of the LEAP node. obfsproxy daemon will redirect incoming traffic to the IP ($dest\_ip) and port ($dest\_port) of a chosen LEAP VPN gateway. 

Listener will use pluggable transport ($transport) and pluggable transport's parameters ($param).

Logs are kept in '/var/log/obfsproxy.log' and are rotated according to '/etc/logrotate.d/obfsproxy'. Default log level is set to 'info', meaning pretty basic information is diplayed about the state of obfsproxy daemon. Logging can be turned completely off or set to another level, in any case no client sensitive information is kept.

obfsproxy puppet module was written so as to be pluggable transport agnostic.

'site\_obfsproxy' is using 'obfsproxy' and sets 'scramblesuit' as pluggable transport.

'site\_obfsproxy' is included if LEAP node's service is 'openvpn' or 'obfsproxy'

A 'obfsproxy.json' was added in provider\_base/services. In there the VPN gateway address is evaluated, which later will be used as $dest\_ip, as well as, the port for obfsproxy daemon and a password for the 'scramblesuit' pluggable transport. These options were also added in 'openvpn.json' too.

###leap\_cli

Two new macros were added in leap\_cli, 'rand\_range' and 'base32\_secret'. Both are used in provider base service json files. The first picks a random port from a certain range for the obfsproxy listener. The second creates a base32 secret for the 'scramblesuit' protocol. These values are stored in secrets.json and should be unique for each obfsproxy instance.

###leap\_webapp

obfsproxy listener connection details are included in a new version of 'eip-service.json'. For every obfsproxy listener there is a hash containing the IP address of the remote obfsproxy server, and for every supported pluggable transport of that server, the port and the paremeters (if any) of the transport.

Since a new version of 'eip-service.json' was introduced, there was a need for a client to list all available versions of a given service definition file. 

For that reason 'configs.json' was introduced and is served via the API too. 'configs.json' lists all available versions for every service definition file.

###bitmask

bitmask client while bootstrapping with LEAP provider will download 'configs.json'. Then bitmask will favor the latest version of each service definition file and will download it. 

Service definition files will be written to disk always with the standard name '$service-service.json' so as to avoid confustion. For example, if provider serves both 'eip-service.json' and 'eip-service-2.json', bitmask will download the second but will save it in the appropriate directory as 'eip-service.json'

Both 'configs.json' and 'eip-service-2.json' use new specs.

bitmask's interface will present 'VPN over obfsproxy' as an extra option that user must enable. This is currently implemented with a checkbox in the VPN status/control area.

As long as user checks the 'VPN over obfsproxy' option and initiates the login procedure, bitmask downloads 'eip-service.json' in order to get the obfsproxy connection details.

Backend makes some preliminary checks on whether obfsproxy can be launched and then launches obfsproxy as socks listener. obfsproxy  

Bitmask uses openvpn command with the --socks-proxy argument, pointing to localhost:port where obfsproxy local instance listens.

Since obfsproxy's socks implementation supports only TCP, VPN connection needs to be TCP as well.

##Considerations / Further improvements


   * Bitmask with obfsproxy enabled will obfsuscate VPN traffic. If bitmask initiates registration or login procedure with a provider, HTTPS traffic to the provider's API will be completely visible to the adversary. As long as bitmask client has already available VPN client certificates ans service definitions files, for example have fetched them in a previous point in the past, it should be able to launch VPN over obfsproxy without any interaction with API (login).
   * There could be a way for the providers to create special bitmask bundles that include obfsproxy and are preseeded with client certificates and configuration. This bundle could then be served to a user via a non-standard channel. This would be useful for users operating in networks with restrictions that would not permit them to even register, login and bootrstrap with a provider.

##LEAP administrator perspective

An obsfproxy instance will be automatically deployed in every LEAP VPN node of a given provider.  Apart from that, LEAP provider's administrators can deploy standalone obfsproxy nodes. 

Standalone obfsproxy nodes can be useful in case an adversary is blocking the VPN node's gateway address or even a larger set of IP addresses used by the provider. A node with obfsproxy listener acts merely as proxy for the end-user: receives the obfuscated traffic from the client and forwards it the provider's VPN gateway. A standalone obfsproxy node has no user data or other sensitive information about the LEAP provider. Thus can be a "disposable" or short-lived node in a completely different environment from the rest of the provider's nodes.

Provider should inform users under what circumstances they should enable VPN  over obfsproxy. Obfuscated VPN traffic will be hard to distinguish and  block but will increase VPN's latency. 

Latency is introduced due to two main reasons:
    a) overhead by the obfuscation of traffic. In fact some pluggable transports, like scramblesuit, modify packets' length and inter-arrival time so as to defend against traffic analysis and flow recognition
    b) in case client uses a obfsproxy node wich is not close the the VPN gateway, round trip time will be augmented

##LEAP end-user perspective

Bitmask user can use VPN over obfsproxy, when they operate within a restrictive network that identifies and blocks VPN traffic. User should be aware that obfsproxy will only hide the fact they use VPN with the LEAP provider, but will not hide the destination IP address. 

User should not try to initiate ordinary VPN connection when in restrictive network, as VPN traffic recognition may lead to permanent block of the LEAP provider's IP address space or other undesired consequences.

User should be aware that there is a tradeoff between stealth communication and low-latency high-bandwidth communication. Traffic obfuscation will inherently increase latency and possibly affect communication's channel available bandwidth, affecting the overall experience.


