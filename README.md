Overview

Amplicon P2P protocol is Skymel’s proprietary one-to-one data transit protocol which possess the following attributes and guarantees:
Each communicating endpoint is uniquely identified by a public-key (access to messages sent to an endpoint are controlled by knowledge of the corresponding private-key). Messages in transit are encrypted.
No other information is required/explicitly transmitted to facilitate communication between two nodes.
Nodes need not share any information regarding their IP addresses, geolocation etc.
By tuning the transit parameters of an established bidirectional communication session, participating nodes can control two key properties of the network route:
Route redundancy (how many different paths exist between the two nodes in real time).
Data package max hops to destination (this limits the maximum possible latency).

In this document we will provide code examples detailing a simple client and server communicating over Amplicon P2P network. For more complicated applications, please wait for a full wiki release or get in touch with us.
Client Example

from skymel.amplicon_network import DataTransitConfigurationError, SessionError
from skymel.amplicon_network import Endpoint, DataTransitConfiguration


remote_endpoint = Endpoint(
public_key="a104d469a3cdfe5cd79763e00773f92819fef0a9707affc76a01f05a39c0ae2a")


try:
   data_transit_configuration = DataTransitConfiguration(
       max_hops=12, max_redundant_routes=5, min_redundant_routes=3)
   session = remote_endpoint.connect(
       data_transit_configuration=data_transit_configuration)
   session.write(b"Hello I am any binary message")
   response = session.read(blocking=True)
   # Access binary payload via {@code response.binary_data_payload}
except DataTransitConfigurationError as dtce:
   # Most likely we cannot find a valid configuration that satisfies all
   # the specified requirements
   print(str(dtce))
except SessionError as se:
   # Likely session timed out during establishment because no path to
   # remote endpoint could be found (most likely because remote endpoint
   # is offline).
   print(str(se))





As shown in the code-example above, we identify the remote endpoint we want to connect to by it’s public_key using:
remote_endpoint = Endpoint(
public_key="a104d469a3cdfe5cd79763e00773f92819fef0a9707affc76a01f05a39c0ae2a")


Please note, that we do not have any information about the private-key of the remote endpoint, or its IP address etc.


Next we specify a DataTransitConfiguration object with desired network transit parameters, which include (but are not limited to):
max_hops
min_redundant_routes, max_redundant_routes


Finally, before sending any data-packets, we try to establish a session with the desired DataTransitConfiguration. The session establishment itself may take a few seconds, however once established the bidirectional data transfer is near-instantaneous (depending on the specified max_hops parameter).




Once the session is established, all the communications to the specific endpoint can be carried out using read, write methods (as shown).


The read method in particular returns a DataPackage object type. The actual binary data payload can be read from the binary_data_payload property.
Server Example

from skymel.amplicon_network import DataTransitConfigurationError, SessionError
from skymel.amplicon_network import Endpoint, DataPackage


# Note you don't need to specify the public_key.
# It's automatically calculated from the private key.
current_server_endpoint = Endpoint(
   private_key="dee960fa17370daafe1447eccb11a1990d8564e5cda6925ad2ee2a7d50fda0ed")


try:
   session = current_server_endpoint.listen()


   while True:
       read_data_package = session.read(blocking=True)
       print(read_data_package.binary_data_payload)
       outgoing_data_package = DataPackage(
                binary_data_payload=b"Response message",
                data_transit_configuration=read_data_package.data_transit_configuration)
       # We can use the incoming read_data_package's data_transit_configuration to 
       # respond back to the client node, without knowing any other details 
       # regarding the client.
       session.write(data_package=outgoing_data_package)
except DataTransitConfigurationError as dtce:
   # Most likely we cannot find a valid configuration that satisfies all
   # the specified requirements
   print(str(dtce))
except SessionError as se:
   # Likely session timed out during establishment because no path to
   # remote endpoint could be found (most likely because remote endpoint
   # is offline).
   print(str(se))



For a server implementation the major steps are outlined in the following sentences. First, we need to create an Endpoint with the private_key instead of the public_key (the public-key is easily derived from the private-key, but not vice-versa), as shown:

current_server_endpoint = Endpoint(
private_key="dee960fa17370daafe1447eccb11a1990d8564e5cda6925ad2ee2a7d50fda0ed")


Next we specify that we are listening for incoming messages, and establish a listening session.


session = current_server_endpoint.listen()


Post session establishment, we enter a perpetual loop within which we listen for incoming messages and write back a binary string (such that the source of the message receives it).


while True:
      read_data_package = session.read(blocking=True)
      print(read_data_package.binary_data_payload)
      outgoing_data_package = DataPackage(
binary_data_payload=b"Response message", data_transit_configuration=read_data_package.data_transit_configuration)
      # We can use the incoming read_data_package's
      # data_transit_configuration to 
      # respond back to the client node, without knowing any other details 
      # regarding the client.
      session.write(data_package=outgoing_data_package)




In this example, all this is performed in a single thread (the main thread). In a production application this loop would involve downstream multiprocessing of read/writes using a dispatch method.


Now, let’s break down the loop shown above into primary components:


We read incoming messages using a blocking read method. The method returns a DataPackage object which contains the following properties:
binary_data_payload - The binary data sent by the client.
data_transit_configuration - The data transit configuration used by the client’s session to reach the current server endpoint. We can re-use this configuration to send messages back to the client, without knowing any other details regarding the client (such as the client’s IP etc).
We can create a new out-going DataPackage  with the received DataPackage’s data_transit_configuration. We can then write using the common server session, and the data-package will be routed to the right client.




