# Unit Hardware Integration Document

## Validity

This document is valid as of:
- cvmain version 3.0.4
- 18 April 2025

## Overview
The VaultGroup locker system comprises a mix of various hardware boards
and the software required to control that hardware. All hardware access
has been abstracted away into an application called "cvmain". "cvmain"
includes a built-in gRPC server that exposes various endpoints that may
be used to control the underlying hardware.

For more information on gRPC, please visit [grpc.io](https://grpc.io)

Application developers only need to integrate with "cvmain". There is no
mandated programming language. "cvmain" has been written on, and tested with
both x84_64 and ARM (raspberry pi) hardware running linux (Raspbian and Ubuntu
are known to work). The gRPC client may run on the Raspberry Pi (RPI) provided by
cellvault, or it could run on a different machine eg. an Android tablet, a Windows
PC with a suitable network connection, etc. 

We have generated gRPC clients for Rust, Java, Kotlin, Dart, Javascript, and Typescript.
Many more languages are supported officially as per the [grpc.io](https://grpc.io) website,
and unofficially (via libraries for your programming language)

Consequently, not all endpoints may be needed for all integrations. For instance,
an old-fashioned keypad and screen are optional input/output mechanisms provided by
VaultGroup, and are suitable for backoffice and more industrial applications. However,
should the application being developed require a more modern UI, the unit may be
equipped with a touch screen or, as previously mentioned, an Android tablet may be used.
In this case the old-fashioned keypad and screen, and their associated endpoints, will not
be needed.

## The .proto file

The gRPC proto file is available on the VaultGroup documentation website

## Building

This is a rust project and works for PC, RPI3, and Android. The easiest way to build
for PC and android is by using the docker build image. For this, you will need an
AWS account with the appropriate perms. Then:

First, if necessary, login to the AWS docker repo using this command (taken from AWS docs
for linux/mac. Google for windows, or an updated command):

```
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 825351766998.dkr.ecr.eu-west-1.amazonaws.com
```

To pull/update the image:
```
docker pull 825351766998.dkr.ecr.eu-west-1.amazonaws.com/cvmain_rs:latest
```

And finally, to run the image:
```
docker run -it 825351766998.dkr.ecr.eu-west-1.amazonaws.com/cvmain_rs:latest
```

All data is in the /Projects folder, including the cloned git code. The repo is ready
for PC and android builds.

## MQTT Services

The system provides an optional MQTT facility, depending on the solution.
For details, please refer to the documentation for the MQTT project.

## Locker Sizes

There are no standard locker sizes. The sizes of lockers depend on customer requirements.
Consequently, labels such as "small", "medium", and "large" make no sense, especially since
customers sometimes request different locker dimensions after a project has gone live. 
Instead, locker sizes in the API are represented as a number from 1 to 1000 (inclusive). 
Larger numbers mean "bigger", and nothing more. A value of 0 means "unknown", and is 
typically used on projects where all lockers have the same size, and therefore 
dimensions do not matter. 

Each number is a code for a locker size for the project in question. These locker sizes will 
not change for the duration of the project. For instance, suppose there are 2 projects: 
ProjectA and ProjectB

ProjectA might have the codes/sizes 100, 500, and 900, where (LxWxH):
100 is a locker with dimensions 30cm^3
500 is a locker with dimensions 10cm x 100cm x 50cm
900 is a locker with dimensions 20cm x 200cm x 50cm

Project B only has one locker size, so it may be assigned the code 500, where:
500 is a locker with dimensions 100cm x 100cm x 200cm
Here, we could also use the code 0, but if we suspect other locker sizes may be required in
future, a number makes more sense

These codes are absolute values within the context of a project only. As seen in the example above,
we may re-use codes between projects. The codes (eg. 100, 500, 900, 432, 876, etc) do not relate to
the dimensions of the locker in any way other than "a larger value code represents a larger locker".

The locker codes and their associated dimensions will be provided to developers, and these may
be hardcoded in the business logic if required.

## Client Startup Guide

The gRPC server must be started before the gRPC client. The server is started automatically when cvmain executes. However,
there is a startup delay, and depending on the hardware being used, this delay may range from several milliseconds to several
seconds.

The delay is most notable on Android solutions where cvmain is packaged into a library that is started by the main
application. In this case, the following sequence is what most developers expect to do:

1. start your Android application
2. Your android application starts cvmain from the VG provided .jar file
3. Create gRPC client and communicate with cvmain

The problem occurs between steps 2 and 3. Specifically, the client at step 3 may be ready to send gRPC calls BEFORE
the server is ready at step 2.

There are 2 solutions:
1. (Recommended) call the get_version() endpoint from the client repeatedly (say, with a 100ms to 500ms delay between calls) 
   until a response is received (if the client and server are on different machines, adjust this time accordingly to compensate
   for network latency). If no response is received after 20 seconds, assume error. 20 seconds is typically sufficient 
   as the server starts within 0.5-2 seconds in most configurations. Feel free to adjust the 20 second value depending on your
   hardware configuration and startup sequence.
2. (Not Recommended) delay your first call for a fixed amount of time. For instance, if you know that the server will be ready
   after 5 seconds, delay your first gRPC call for 5 seconds.

The first approach is preferred because you will be able to continue with gRPC calls within, typically, 100ms of the server
starting. If you used a fixed delay, your startup time may be considerably longer.

The following is a pseudocode function:

```
function boolean is_server_running(client) {
    client.set_timeout(100) //100ms connect/read timeout
    
    var timer = new_timer_ms(20000) //create a timer that times out after 20000 millis or 20 seconds
    while (!timer.timed_out()) {
        //make an api call
        var result = client.get_version()
        if result.is_timed_out() {
            continue;
        }
        return true;
    }
    
    return false;
}
```

## Basic Messages

### Basic Response

Includes details of success or failure of the request. This is primarily for future
extensibility. Unless otherwise specified, it is safe to detect success/failure using the
standard mechanisms provided by the gRPC platform.

```
message BasicResponse {
    bool success = 1;
    string errMsg = 2;
    int32 code = 3;
}
```

A general response provided when no special data needs to be returned

```
message GeneralResponse {
    BasicResponse resp = 1;
}
```

### RPC Endpoints

#### Get Software Version

Returns the cvmain version number

```
  rpc get_version(google.protobuf.Empty) returns (GetVersionResponse);
```

```
message GetVersionResponse {
  BasicResponse resp = 1;

  //will contain the software version number
  string version = 2;

}
```

#### Toggle Buzzer

Turns buzzer on for a user specified duration

```
rpc toggle_buzzer(ToggleBuzzerRequest) returns (GeneralResponse);
```

```
message ToggleBuzzerRequest {
  //the number of millis for which the buzzer must be sounded
  uint32 duration_millis = 1;
}
```

#### Lock a locker

locks the specified locker
```
rpc lock_locker(LockRequest) returns (GeneralResponse);
```

```
message LockRequest {
  //the locker number to locker. The first locker is always 1. The last locker can be determined by
  //a call to get_locker_map()
  uint32 locker_num = 1;
}
```

#### Unlock a locker

unlocks the specified locker
```
rpc unlock_locker(LockRequest) returns (GeneralResponse);
```

See lock_locker() for the LockRequest data type.

#### Retrieve the date and time

retrieves the date and time from the device
```
rpc get_rtc(google.protobuf.Empty) returns (GetRtcResponse);
```

```
message GetRtcResponse {
  BasicResponse resp = 1;

  //the date+time in rfc8601/3399 format, with timezone information
  string datetime = 2;
}
```

#### Set the date and time

sets the date and time on the device. Note that this is periodically synced
automatically to UTC, so it is not recommended this endpoint be used directly

```
rpc set_rtc(SetRtcRequest) returns (GeneralResponse);
```

```
message SetRtcRequest {
    //the date+time in rfc8601/3399 format, with timezone information
    string datetime = 1;
}
```

#### Clear the LCD

Clears the entire LCD screen. This is for the LCD screen connected to the master board only.

```
rpc lcd_clear_screen(google.protobuf.Empty) returns (GeneralResponse);
```

#### Clear 1 line of the LCD

clears the specified line of the LCD screen. This is for the LCD screen connected to the master board only.

```
rpc lcd_clear_line(LcdClearLineRequest) returns (GeneralResponse);
```

```
message LcdClearLineRequest {
    //the number of the line to clear. Lines (rows) are 0-3, inclusive
    uint32 line_num = 1;
}
```

#### Write data to the LCD

writes some data to the LCD screen. This is for the LCD screen connected to the master board only.

```
rpc lcd_write_data(LcdWriteDataRequest) returns (GeneralResponse);
```

```
message LcdWriteDataRequest {
  //0-3
  uint32 row = 1;

  //0-19. -1 means auto-center data in row.
  int32 col = 2;

  //data to write (no more than 20 chars if starting from column 0)
  string text = 3;
}
```

#### Retrieve locker map

Retrieves the mapping for the unit. Every vault comprises multiple columns, controlled
by slave boards. Every slave is uniquely numbered, starting from 0, and can have a maximum
of 15 (for a total of 16 slaves.) Each slave controls a maximum of 6 lockers in the column.
Since locker sizes may vary, some slaves will control 6 lockers, others will control 3
lockers, etc.

A locker map represents the number of slaves, the number of lockers controlled by each slave, 
and total lockers configured for a particular vault.

```
rpc get_locker_map(google.protobuf.Empty) returns (GetLockerMapResponse);
```

```
message GetLockerMapResponse {
  BasicResponse resp = 1;

  //an array of the number of lockers in each slave (not necessarily column since we've had products where
  //multiple slaves are used in a single column eg. if a 10-locker column is required)
  repeated uint32 lockers = 2;

  //the total number of lockers in the system. This is just the sum of all integers in the array
  uint32 num_lockers = 3;
}
```

#### Notify the system of a duress

Triggers a duress as per the user's request. For instance, the user may enter a special code on the
keypad to have this duress triggered. The underlying action is hardware specific. For instance, on a
RPI, this typically activates a GPIO pin. 

The hardware specific integration is handled by a different
application via an integration layer and can be made to do just about anything (eg. we could send a
message to a rabbit server or call a webhook or something if that's required). This can be
useful, for instance, for users to covertly request assistance in the event of a robbery.

```
rpc trigger_user_duress(google.protobuf.Empty) returns (GeneralResponse);
```

#### Send Audit Message

Allows higher level user applications to take advantage of the vaultgroup auditing facility.
User log messages will be mixed in with vaultgroup messages, but in a private code range allowing
for easy filtering. It is not required that this endpoint be used. Users are free to have their own
logging facilities independent of VG. Should users wish to piggy-back off the vault-group
audit platform, they will need to contact us to provision a facility whereby they can receive
their audit messages.
```
rpc user_audit(UserAuditLogRequest) returns (GeneralResponse);
```

```
message UserAuditLogRequest {
  uint32 code = 1;

  /**
    Valid values are "info", "warning", "error", "fatal" only. All other values will result in an error.
   */
  string level = 2;

  /**
    An optional string. May be used to indicate the source of the error
   */
  string facility = 3;

  /**
    The error message, no more than 1024 bytes
   */
  string description = 4;

  string priority = 5;

  string app = 6;
}
```

#### Send SMS

Submits an SMS for transmission. This will be transmitted via the VG server. Units
will require correct permissions on the server to use this facility, so contact
VG beforehand

```
rpc send_sms(SendSmsRequest) returns (GeneralResponse);
```

```
message SendSmsRequest {
  /* cellphone number */
  string cell_num = 1;

  /* sms data to send */
  string msg = 2;
}
```

#### Authentication token retrieval

Retrieves the authentication token used to log in to VG services. This token
is automatically obtained by the cvmain application. It is made available to
application developers should they wish to access VG server facilities directly
from units. This is generally not recommended, but the facility is available if
required.

```
rpc get_auth_token(google.protobuf.Empty) returns (GetAuthTokenResponse);
```

```
message GetAuthTokenResponse {
  BasicResponse resp = 1;

  /**
    Gives access to a copy of the JWT token used for authentication by this app.
    This allows the client to access VG APIs if required. An empty string i.e. ""
    in a success response means no token is currently available
   */
  string token = 2;
}
```

#### Set state on multistate slave boards

A command to set the state on slave boards running multistate or similar firmware.
this command is NOT available on regular slaves. Multistate slaves include modified
logic such that the door lock button can be used as a door open+lock button to
repeatedly access a locker without further keypad input from the user. On
completion of this cycle, the user cancels the operation from something like
the keypad, by altering the internal slave state logic.

Do not use this endpoint unless your solution uses multistate slaves. Most solutions
do not require this functionality

```
rpc set_locker_state(SetLockerStateRequest) returns (GeneralResponse);
```

```
message SetLockerStateRequest {
  /**
    the locker to access. The first locker is 1
   */
  uint32 locker_num = 1;

  /**
     the state to set the locker to. Valid values are:
     LS_OPEN = 0
     LS_LOCKED = 1
     LS_READY_OPEN = 2

   */
  uint32 state = 2;
}
```

#### Ping

a simple endpoint that can be called to see if the server is operational

```
rpc ping(google.protobuf.Empty) returns (GeneralResponse);
```

#### Retrieve current locker states

returns the states for every door (open/close based on the door switch) and
lock (locked/unlocked)

```
rpc get_locker_states(google.protobuf.Empty) returns (GetLockerStatesResponse);
```

```
message GetLockerStatesResponse {
  BasicResponse resp = 1;

  //an array of integers where -1 means not initialized, 0 means closed, 1 means open.
  //each number is for 1 locker (eg. if there are 20 lockers, there will be 20 items
  //in the array)
  repeated int32 door_map = 2;

  //each item is for 1 locker (eg. if there are 20 lockers, there will be 20 items
  //in the array)
  repeated LockerStateResponseMessage locker_map = 3;
}
```

```
message LockerStateResponseMessage {
  bool initialized = 1;
  LockerStateMessage state = 2;
}
```

```
message LockerStateMessage {
  /**
   * See comment in SetLockerStateRequest
   */
  uint32 state = 1;
  
  /**
   * The size of the locker. See the section on
   * locker sizes for an explanation
   */
  uint32 locker_size = 2;
}
```

#### Retrieve slave firmware versions

Returns the version number for each slave board. The system will
not start if the wrong slaves and/or locks have been configured. Currently,
slave firmware with major number "1" is for regular use. Slave firmware with
major number "10" is for multistate boards. The underlying "cvmain" application
will fail to start if invalid slave firmware versions are detected.

```
rpc get_slave_firmware(google.protobuf.Empty) returns (GetSlaveFirmwareResponse);
```

```
message GetSlaveFirmwareResponse {
  BasicResponse resp = 1;

  //a value eg. "1.5", where "1" is the major and "5" is the minor
  repeated string firmware = 2;
}
```

#### Set the LED for a locker

Every locker has an optional LED that can be set to off, red, green, or orange. When
a door locks, the LED is automatically set to red. When a door unlocks, the LED is
automatically set to green. If a user attempts to engage the lock via the door button
while the door is still open (as determined by the door microswitch), the LED will fail
to lock and the LED with temporarily flash orange.

This command allows the user to explicitly set the LED color. The operations above will
still be affect the LED color.


```
rpc set_led(SetLedMessage) returns (GeneralResponse);
```

```
message SetLedMessage {
    uint32 locker_num = 1;
    
    //0 for off, 1 for red, 2 for green, 3 for orange
    uint32 color = 2;
}
```

#### Reboot

This commands allows a remote device to reboot the system. The actual reboot
is performed by an external application or script, and the entire OS is
rebooted.

The command requires a code/token. Currently, the code must be calculated as follows:

The token is made up of data in the text file ${base_dir}/reboot_code, combined
with the utc time in seconds since the epoch rounded down to the last 100
seconds. A dash (-) separates these values. The sha256sum is then taken
of these values, and the lowercase hex encoded value is the hash.

For instance:
```
x=get_file_contents("${base_dir}/reboot_code")

//Suppose the unix time (utc) is t=16045968777. This must be rounded down to
//16045968700. The calculation is: t = t - (t % 100) 
t = get_utc_epoch_time()
t = t - (t % 100) 

the value to hash is: v="${x}-${t}"
token = sha256sum(${v})
```

This system ensures the reboot hash changes every 100 seconds, and the
reboot code is never shared between systems

```
rpc reboot(RebootRequest) returns (GeneralResponse);

message RebootRequest {
  string code = 1;
}
```

### Register MQTT integration

Depending on the solution, the platform may run a
MQTT service for server-to-client messages. If in use,
this endpoint allows a non-cvmain application to
register for messages. When an MQTT message is received,
the details of this integration allow the message to be
directed to the correct destination.

Every name must be unique. Names beginning with "vg:" are
reserved for Vaultgroup applications.

If this call is made more than once with the same integration
name, the integration details are updated only i.e. it is safe
to register an integration more than once.

```
//Used to register an mqtt integration by an external app. Available as of
//1.0.2
rpc register_mqtt_integration(RegisterMqttIntegrationRequest) returns (RegisterMqttIntegrationResponse);

//used to register an external integration with the mqtt service
message RegisterMqttIntegrationRequest {
  //the name of the integration
  string name = 1;

  //integration details (see message definition for more)
  MqttCommsIntegration integration = 2;
}

//basic response message after registering a mqtt integration
message RegisterMqttIntegrationResponse {
  BasicResponse resp = 1;
}
```

### Unregister MQTT integration

Removes the registered MQTT integration, if it exists.

```
//Used to unregister an mqtt integration by an external app. Available as of
//1.0.2
rpc unregister_mqtt_integration(UnregisterMqttIntegrationRequest) returns (RegisterMqttIntegrationResponse);

//message to unregister an integration
message UnregisterMqttIntegrationRequest {
  //the name of the integration, specified when registering
  string name = 1;
}
```

### Send notification

Allows a third party app (eg. A customer) to send a non-vaulgroup
message, should they need to. The message will be sent via the notification
system, using the notification message format.

```
//allows a third party app to send a non-vg notification message.
rpc send_notification(NotificationMessageRequest) returns (BasicResponse);

 /// the first string is the address:port to which we must bind. The second
 /// is the address:port to which the message must be sent
message IntegrationItem {
  string bind_addr = 1;
  string send_addr = 2;
}

//a notification message
message NotificationMessageRequest {
  //the type of notification message
  string msg_type = 1;
  repeated KVPairItem vals = 2;

  //if specified, the address to which the message must be sent.
  IntegrationItem integration = 3;
}
```

### User event callbacks

The application implements a callback system for detection of events such as keypad presses,
door lock/unlock, door open/close.

Notification messages are passed up via UDP. The UDP host and port to which callbacks are sent may
be configured in the cvmain configuration file. Every UDP packet contains one event
only.

By default, the callback system sends data to 127.0.0.1:5555. To view events without
writing code, the simplest way is to use netcat like so (on Linux):

```
nc -l -u 5555
```

All notification messages have the same data structure:

```
{
    "type": "string",
    "vals": [
        {
            "k": "some string key",
            "v": "some string value"
        }
    ]
}
```

#### Door opened notification

```
    {
        type: "door_opened",
        vals: [
            {
                "k": "locker",
                "v": "5"        
            },
            {
                "k": "offset",
                "v": "[0:4]"
            }
        ]
    }    
```

Where: 
- val: locker refers to the locker number (counting from 1)
- val: offset refers to the slave/locker (slave counts from 0, locker counts from 0)

#### Door closed notification

This is the same as the door opened notification, but the type is "door_closed"

#### Door locked notification

This is the same as the door opened notification, but the type is "door_locked"

#### Door unlocked notification

This is the same as the door opened notification, but the type is "door_unlocked"

#### Key pressed notification

Indicates a key was pressed on the keypad attached to the master board

```
    {
        type: "key",
        vals: [
            {
                "k": "value",
                "v": "49"        
            }
        ]
    }    
```

Where "49" is the ASCII value 49, or char '1'. All values are provided in ASCII. Valid
(ASCII) values are the characts 0-9, * and #. These are the only keys supported by the
physical keypad.

#### Duress detected

Indicates the user triggered the duress endpoint. This is a generic duress message

```
    {
        type: "duress",
        vals: [
            {
                "k": "dt",
                "v": "2022-01-01T00:00:00Z"        
            }
        ]
    }    
```

Where:
    - "dt" means datetime
    - the datetime is provided in RFC8601/RFC3399 format

The full format of duress messages looks like so:

```
{
    "type":"string",
    "vals":[
        {
          "k": "dt",
          "v": "2024-01-01T00:00:00Z"
        },
        { "k":"details",
          "v":"string"
        }
    ]
}
```

Supported types are:
- duress - indicating a general duress message. No details are provided.
- locker_tamper - indicating that locker tampering has been detected. The details string will be "locker=xxx" where xxx is the locker number
- locker_tilt - indicating that the locker system is being tilted eg. someone is trying to push it over
- locker_vibration - indicating that unexpected vibrations are detected and may indicate a break-in attempt (eg. using a drill or other power tools)

#### RFID card

An RFID card was detected and read, matched against the internal database, and is valid.

```
    {
        type: "rfid_card",
        vals: [
            {
                "k": "card",
                "v": "1234567890",           
            },
            {
                "k": "dt",
                "v": "2022-01-01T00:00:00Z"        
            }
        ]
    }    
```

Where:
    - The value for the "card" field is the rfid card value
    - the value for "dt" is the datetime in rfc8601/3399 format

#### Unknown card detected

Notification sent when an RFID card was read successfully, but it could not be matched
against a valid value in the internal database.

Data structure is same as for "RFID card", but the type is "rfid_unknown"


## Configuration

At startup, the system requires a configuration directory as a
parameter. All configuration items for the application are contained
within this directory. Switching between testing environments and
production environments is also a simple case of restarting the application
with a different configuration directory.

The main configuration items are documented below.

### config.json

This is the main configuration file. It is mandatory.
An documented example is shown below

```
{
  //the type of communication system to use. This may be "net" or "serial"
  //"serial" connects to a RS232 port (physical, virtual, USB->serial, etc).
  //"net" connects via TCP, to a device that must provide
  //a TCP->[RS232/other] protocol conversion. "net" can be used
  //with android integrations to allow for WiFi->RS232 conversions, when
  //coupled with the VG WiFi->RS232 passthrough device. Alternatively, it may
  //be used for testing on a laptop by using a tool like "socat" to provide
  //a TCP->[virtual/physical RS232] passthrough.
  "comms": "net",
  
  //The communications latency multiplier. When using "net", the 
  //receive latency on messages may be greater, depending on the network
  //configuration (for instance, when using WiFi->RS232, there's now
  //WiFi latency, RS232 latency, and the latency associated with converting
  //WiFi data to RS232). This affects timing. This field allows for the timing
  //to be adjusted. When using "serial", this should be set to 1. When using "net"
  //the WiFi to RS232 passthrough, it is typically set to between 3 and 5. 
  //When using a laptop and something like socat, a value of 1-2 usually works,
  //assuming "socat" is running on localhost. Should a value of 10 or more be 
  //required, it is usually a good indicator that your network configuration should
  //be adjusted. 
  "comms_rx_multiplier": 1,
  
  //configuration for the duress integration. Duress messages may be
  //triggered from any source (eg. via a button connected to a GPIO, via a
  //special code entered on a keypad, etc). Regardless of the source, 
  //duress messages are converted to files placed in a directory. When cvmain
  //sees one of these files, a duress event is generated and sent to the server.
  //By using files as the integration, it allows for duress messages to be triggered
  //from any source, including a client application. Hypothetically, a client may
  //use a camera combined with AI to identify a "duress" situation using a proprietary
  //application, then simply create an appropriate duress file. cvmain will
  //identify the file and take care of the rest.
  "duress": {
    //the directory where cvmain will look for duress files
    "cache_dir": "var/duress",
    
    //the name of the file created by a hardware trigger. For
    //instance, if a panic button is pressed, the system may
    //detect it and create this file to indicate a duress
    "hw_duress_trigger_file": "duress",
    
    //the name of the file created by a user/software tigger. For
    //instance, if the user types a special code on the keypad to
    //trigger a duress, the duress system should create this file.
    "user_duress_trigger_file": "user_duress"
  },
  
  //Certain operations are performed by using external applications
  //or scripts. These are always found in the "bin" directory within
  //the main config directory
  "external_bins": {
    //This sets the date and time on the platform. It may be adjusted,
    //depending the the platform. For instance, when using a raspberry pi,
    //this script changes the date/time on the system. When used on
    //a PC during testing, this script does nothing because the PC time
    //is usually managed by the OS. Similarly for Android, this script does
    //nothing since Android manages the date/time.
    "datetime": "set_date.sh",
    
    //The script/app to reboot the system. Used on a RPI,
    //but it is usually set to an empty script for Android and
    //developement systems
    "reboot": "reboot.sh"
  },
  
  //how frequently the system must refresh its
  //settings from the server. This is particularly useful
  //when working with rfid cards. The value is specified in ms.
  "get_settings_time": 3600000,
  
  //cvmain integrates with external system by using a gRPC server.
  //this is the configuration for that server
  "local_server": {
    //the host and port to which to bind. By default, the system binds to
    //localhost i.e. 127.0.0.1. This is usually good for production, but
    //inconvenient for development. Consider binding to 0.0.0.0 if
    //external connections are required (eg. if you want to test with something
    //like BloomRPC or Postman running on your PC)
    "bind_addr": "127.0.0.1:7777"
  },
  
  //when true, additional debug messages are displayed
  "log_debug": false,
  
  //The column mapping i.e. the number of lockers in each
  //column. eg. 5,6,6 means there are 3 columns with 5 lockers,
  //6 lockers, and 6 lockers, respectively. Note that currently,
  //and depending on your configuration, a maximum of 16 columns
  //may be specified. On each column, there is a maximum of 6 lockers.
  "mapping": [
    6
  ],
  
  //for mqtt. Settings like mqtt values are obtained from
  //the server as part of the system settings, and kept in the
  //"mq" directory. All mqtt configurations are managed in
  //this "mq" directory as well.
  //
  //Note: if external_integrations is true and msgs_via_notifier
  //is true, it is possible that 2 messages will be sent. If both
  //are false, no message will be sent.
  "mq": {
    //when true, mqtt messages for external integrations can be
    //sent via the IP addresses specified when the integration
    //was registered.
    "external_integrations": false,
    
    //mqtt messages are just general notification messages.
    //when using an external integration, messages may be
    //sent via that integration. However, for simplicity, a client
    //may want to just sent their mqtt messages via the notification
    //system. Setting this to true sends client mqtt messages
    //via the standard system notifier.
    "msgs_via_notifier": true
  }
  
  //when the "comms" field is set to "net", this
  //configuration block is used
  "net": {
    //the ip/port to which cvmain will connect
    "url": "192.168.64.3:5433"
  },
  
  //configuration of the notification server, for
  //sending notification messages
  "notification_server": {
    //the address where the notification server will bind
    "bind_addr": "127.0.0.1:5554",
    
    //the address where notification messages will be sent
    "send_addr": "127.0.0.1:5555"
  },
  
  //configuation for the vaultgroup server
  "remote_server": {
    //address of the vaultgroup server
    "base_url": "https://saas.vaultgroup-cloud.com"
  },
  
  //rfid card configuration
  "rfid_cards": {
    //where rfid card scans will be found.
    //card scans are handled by an external application,
    //allowing for a variety of rfid hardware integrations
    //to be used.
    //
    //When a card is scanned, the file "rfid_card" is
    //created an placed in this directory. "rfid_card"
    //is a text file with exactly 2 lines. line1==the rfid
    //card value, and line2==the date/time in RFC3339 format.
    "cache_dir": "var/cards"
  },
  
  //how frequently, in ms, to refresh the date/time from the server
  //to ensure the date/time on the system is correct.
  "rtc_refresh_time": 1800000,
  
  //when the "comms" field is set to "serial", this
  //configuration block is used
  "serial": {
    //the path to the serial port
    "port": "/dev/ttyS0"
  },
  
  //the VG time server
  "timeserver": {
    //address of the VG time server. Note that this is HTTP, not HTTPS.
    //Should the date/time be sufficiently incorrect, a HTTPS handshake cannot
    //be negotiated, hence HTTP is used. Please do not set this to HTTPS.
    "url": "http://timeserver.vaultgroup-cloud.com/now"
  },
  
  //VG provides 2 locking mechanisms, an "in-house" lock,
  //and an off-the-shelf lock. When true, the system is configured to
  //use the in-house lock. When false, the off-the-shelf lock is used
  //instead
  "use_cv_locks": true,
  
  //when true, the code is adapted to use multi-state slave firmware.
  //This is only required if the product in question uses multi-state
  //slave boards. Almost all customers will set this to false. If you 
  //are not using multistate slave firwmare and this is set to true, the
  //system will not start correctly, and complain about running an incorrect
  //slave version.
  "use_multistate_slave": false,
  
  //One of the VG hardware options is a numeric keypad that connects
  //to the VG master board. If your hardware solution uses this keypad,
  //set this option to true. If not, leave this option as false for a 
  //performance boost.
  "use_keypad": false,
  
  //defaults to true. If false, the rtc will not be synchronized. Useful when
  //using eg. an android tablet and we have no control over the RTC. 
  "use_rtc": false,
  
  //defaults to false. If true then absence of internet connectivity will be assumed.
  pub use_standalone_mode: false
    
  //for the tamper system, how long a door is allowed to be opened
  // after an unlock command is issued. applies only to non-cv locks.
  // defaults to 30000ms
  pub tamper_max_door_open_time_ms: 30000

}
```

### auth.json
A file containing the username and password, to login to the
vaultgroup server. This is configured at registration time only.

### reboot_code
In order to safely perform a remote reboot, this contents of this file are
required.

### log_conf.yaml
The log configuration file. Logging is handled by the log4rs project,
available [here](https://docs.rs/log4rs/latest/log4rs/)



