# VaultGroup SaaS Documentation

The Software as a Service (SaaS) platform allows third party integrators to 
create their own solutions that integrate with VaultGroup (VG). Features of this platform
and service include:

- Customers have full control over their applications
- Customers have control of the VaultGroup hardware via a basic gRPC integration
- Customers may create  proprietary solutions, including solutions running on their own hardware if required
- Customers are free to use their preferred programming languages, frameworks, even operating system should they choose to do so

## The Server Platform

All VG units connect to the VG server platform. Customers with accounts on the platform may access
their accounts at https://web.vaultgroup-cloud.com . Additionally, a REST API is available to allow
customers programmatic access. Please find documentation in the server/ directory.

## Unit Access

VG has created a Linux-based application called "cvmain" that serves as the communications proxy
between "business logic" created by customers, and the VG unit hardware. 
Customers are free to create  their business logic and interface with the VG hardware by
connecting to cvmain.

### cvmain

cvmain is the proxy that allows customer business logic to communicate with VG hardware.
cvmain includes a gRPC server. Customers may communicate with cvmain by implementing a
gRPC client in the programming language of their choice. The "unit/" directory includes
cvmain documentation, as well as a **service.proto** file for generating the gRPC client. For
more information about gRPC, please visit https://grpc.io

### cvmain Client Code

Note that as with all gRPC projects, it is not necessary to write a client from scratch. Simply use
the supplied service.proto file to generate gRPC client code in your language of choice. We have successfully
generated code for Java, Android (Java/Kotlin), Rust, Javascript/Typescript. Support for many more
languages is available. Please visit https://grpc.io

### cvmain Environment

cvmain is a linux application and has been tested with various linux configurations. The code
also runs on MacOS with limited communications options. The code was not designed for use on
Windows, and has not been tested in a Windows environment.

If using VG hardware, cvmain typically runs on a Raspberry Pi 3 or similar device. The
Raspberry Pi is not a requirement. cvmain has been run on x86 laptops, ARM laptops, Macbook Pro,
Raspberry Pi, and Android phones and tablets. It is possible to run cvmain on a 
Raspberry Pi for instance, and have a customer's business logic run on a Windows PC 
that connects to cvmain on a Raspberry Pi via gRPC. This allows the customer to write
business logic, for instance, in C# and Windows, but still interface with the Linux-based
cvmain. Customers are free to include their software on the Raspberry Pi if they choose.
Customers may choose to run a custom operating system on the Raspberry Pi (or similar), but
with the cvmain software. This is also acceptable. No limitation is placed on
programming language. The only requirement is the customer's business logic must be
able to communicate with the gRPC server included within cvmain.

### cvmain Android

cvmain runs successfully on Android devices. Testing has been performed on Android 10+.
VG has developed an Android/Java library to easily create allow for cvmain to be run
from within an Android project.

Please see https://github.com/vaultgroup-sa/cvmain-android-sample for documentation,
as well as sample code.

## Simulator

A simulator has been developed to allow for easy development without the need for physical
hardware. The simulator runs in a docker container and performs all the basic functionality
of a physical locker system.

Please see the "simulator/" directory for documentation. 


## Sample Project

Visit https://github.com/vaultgroup-sa/cv-vaultgroup-sdk for a sample project that shows how
code can be integrated with the VG SaaS platform. The code is free for use and modification.

