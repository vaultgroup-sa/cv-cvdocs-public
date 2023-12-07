# cvmqtt Documentation

## Overview

VG includes an optional MQTT integration, depending on the solution. This application,
called cvmqtt, processes and distributes MQTT messages received from the server. 

Note that third party developers never directly communicate with cvmqtt. 
All communication is handled via cvmain for simplicity. The cvmqtt can be considered 
a "module" that provides MQTT services. cvmqtt does not contain a gRPC server, or any
direct communications facilities. Everything is done via cvmain.

The purpose of this document is to explain how the VG MQTT system works, and how
third-party developers may use the system.

## Requirements

This application connects to the VG enterprise platform using the authentication 
token obtained by cvmain. Settings retrieved by cvmain are saved in the "mq" 
subdirectory of cvmain's main configuration directory. This application references
that directory as well.

Therefore, a registered, running copy of cvmain is required for cvmqtt to function.

## Integrations

cvmqtt uses "integrations" from other applications. Integrations
each have a unique name, and they allow the cvmqtt to forward MQTT
messages received from the server to the appropriate destination application,
via the respective integration.

Integrations are registered with cvmain, via the gRPC server. Please refer to
the cvmain documentation for details on registering and unregistering integrations.


## Integration names and message names

Integrations offer a means for other applications to receive mqtt messages. All
MQTT messages will be received from the same MQTT server, but there needs to be
a way for these messages to reach the correct application. Integrations allow for
this.

### Integration names

Every integration has a name. Every name must be unique. In this way, every
integration can be uniquely identified. For instance, the name may be called
"int1". The integration name **should** not contain a dash (-), though this
rule is not enforced in code.

### Message names

Every message has a name. For instance, suppose there is a message called
"get-settings". The problem with a name like "get-settings" is another
integration may use it. This creates a message-routing conflict.

Therefore, the message name is always <integration-name>-<message-name> eg. 
"int1-get-settings". Everything before the first dash i.e. "int1", is the
integration. Everything after the first dash i.e. get-settings, is the  
message.

When cvmqtt receives a message like int1-get-settings, it knows to route that
message to the "int1" integration. Therefore, the correct application will 
always receive the correct message.

### Reserved names

All integration names starting with "vg:" are reserved for vaultgroup
applications. Clients may not use these names. For instance, 
vg:upgrader-get-settings is the get-settings message for the 
vaultgroup upgrader app. Any message that does not start with a "vg:"
is meant to be directed to a third-party developer app.


## Message distribution

Messages will always be sent via the notification system available in cvmain. Refer
to the cvmain documentation for more info