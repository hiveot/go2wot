# go2wot Web-of-Things Concentrator

go2wot is a concentrator for IoT devices. It supports a myriad of IoT devices and serves these as as WoT (W3C Web Of Things) Things using WoT standard protocols. go2wot works in combination with IoT agents that convert IoT protocols to a WoT compatible interface.

go2wot can be run as a standalone server or embedded as a library in a golang application.

go2wot is inspired by go2rtc which is a concentrator for video streams. Is is a spinoff from the HiveOT's hub.

## Project Status

Nov 2025: Kickoff, extract components from the hiveot hub.

Roadmap:

1. Design documentation
2. Configuration management
3. IoT device agent plugins
4. Auth
   4.1 Server certificate management
   4.2 Consumer authentication
   4.3 Consumer authorization
5. WoT Consumer protocols
   4.1 WoT http protocol
   4.2 WoT websocket protocol
   4.3 WoT MQTT protocol

## Audience

This project is aimed at software developers and system implementors that are working on secure IoT solutions. HiveOT users support the security mandate that IoT devices should be isolated from the internet and end-users should not have direct access to IoT devices. Instead, all access operates via the Hub.

## What is go2wot

go2wot is a golang library and runtime that concentrates access to IoT devices via a single point of entry using the standard WoT application protocols. It maps various IoT device protocols to the WoT standard and unifies authentication, authorization and discovery for these devices. It can be embedded into an application or can run as a stand-alone WoT protocol runtime.

go2wot can be extended through hooks (when used as a library), with plugins, or both.

go2wot uses hooks for consumer authentication and authorization, and offers the ability to listen in on TDs, properties, events and actions messages. One example is to support logging and monitoring of IoT messages.

go2wot agents are plugins that translate between WoT and 3rd party IoT protocols. The agent defines device TD's, publishes property and event updates, and forwards actions for the devices it manages using the native protocol. As such, it provides a collection of WoT Things for these devices and can be used as if it is the Thing itself. The go2wot runtime connects to these agents through one of the fast transport methods such as UDS or websockets. It collects the available Things from the various agents and makes them available to its consumers.

The go2wot concentrator supports multiple WoT protocols for consumers to connect with, such as HTTPS/SSE, WebSocket, and MQTT message bus protocols.

![go2wot overview](docs/go2wot-overview.jpg)

go2wot is not a home automation solution. It does not provide a user interface or any IoT application services. Instead it is intended to be used to build automation solutions.

go2wot is also not just a proxy as it knows about the IoT devices it is connected to and adapts their TD. It modifies the TD ID, forms and auth to point to itself, so devices are accessed via the runtime.

## About HiveOT

Security is big concern with today's IoT devices. The Internet of Things contains billions of devices that when not properly secured can be hacked. Unfortunately the reality is that the security of many of these devices leaves a lot to be desired. Many devices are vulnerable to attacks and are never upgraded with security patches. This problem is only going to get worse as more IoT devices are coming to market. Imagine a botnet of a billion devices on the Internet ready for use by unscrupulous actors.

The goal of HiveOT is to support a way to improve security of IoT devices, mainly by isolating them from the rest of the network and providing a single secure endpoint.

The secondary goal of HiveOT is to simplify integration and usage of IoT devices by providing a single consist standardized way of interacting with all IoT devices including authentication, authorization, directory, history and other capabilities.

HiveOT is based on the [W3C WoT TD 1.1 specification](https://www.w3.org/TR/wot-thing-description11/) for interaction between IoT devices and consumers. It aims to be compatible with this standard.

Integration with 3rd party IoT devices is supported through the use of IoT plugins. These plugins translate between the WoT protocol and 3rd party IoT protocols, interacting using properties, events and actions.

The HiveOT Hub uses go2wot as its runtime and adds a digital twin model, service launcher, history, dashboard, and other features such as bridging in a single package.

## IoT Protocol Agent Plugins

### Introduction

IoT protocol agent plugins translate between a standard WoT protocol and a native IoT protocol. They are adapters that can also be used on their own to provide WoT compatible access to native protocols.

Agent plugins run out of the box with their default configuration. If neccesary they can be configured using yaml configuration files or by sending it a configuration update.

Q1: Do agent plugins expose a server? Isn't this a security risk?

The original HiveOT hub uses a reverse connection so agents don't need to run a server. This has two benefits, it is more secure and it reduces the resources used. The downside is that it is non-standard and the agent cannot be reused elsewhere.

go2wot's approach is to separate the server need from the agent main job. The main job is to translate between the IoT native protocol and WoT protocol. Agents can implement hooks that allows the go2wot messaging wrapper to be used for communication so agents don't have to.

The go2wot messaging wrapper exposes a WoT compatible server using https, mqtts, or unix domain sockets as transport protocol, depending on its configuration. It can also support reverse connections instead of running a server, which at this time is non standard.

Note that agent plugins can be written in various languages so a messaging wrapper must be available for each language. As a minimum, https server must be supported.

The go2wot messaging wrapper provides hooks for authentication of incoming connections. The included default implementation supports jwt/paseto bearer tokens validation which can be replaced by application specific authentication.

The go2wot messaging wrapper must be provided a TLS certificate, a corresponding x509 CA certificate and the private key used to create the certificate. Generating this certificate is out of scope of go2wot.

Essentially, Agent plugins are concentrators for the devices they manage. As such they can make extensive use of the go2wot library to manage Things.

### The roles of an Agent

A go2wot agent implements the following roles:

1. Serve a WoT compatible protocol to enable consumers to connect, authenticate and receive requests.

Consumers can interact with an agent as if it is the Thing itself. Since agents support multiple devices, it can act as the Thing for any of the supported devices.

2. Create and publish a TD for each of the devices it represents.

The WoT standard not only specifies how consumers interact with a directory service, but also how Things can write their TD to the directory. The agent will need to discover the location of the directory and write the TD of each device it manages.

The directory location can be specified in the agent configuration or discovered using the WoT discovery specification.

Agents MUST include the agent ID as part of the Thing ID of published TDs. The go2wot concentrator uses this to identify the agent to relay Thing messages to.

3. Handle requests for device operations as defined in the TD.

The consumer needs to provide a valid authentication token that identifies the consumer in order to connect. Connected consumers use the messaging protocol to publish requests and subscribe to property and event updates. The go2wot library provides the boilerplate code for handling requests and subscriptions.

While agents can be configured to accept connections from consumers, their main purpose is to work with go2wot's concentrator. The concentrator uses go2wot specific actions to manage the agent's configuration and running status.

## Directory

Agents publish the TDs of the devices they manage to the available directory service.

A directory service itself is not part of go2wot but is included in the HiveOT Hub. Users of go2wot can specify the location of a Directory service, whether that is of the HiveOT Hub or a 3rd party directory service.

The go2wot library includes the facility to discovery a directory, publish a TD and query the directory.

If go2wot is used as part of the HiveOT ecosystem then the Hub directory service is available to write a TD to. The go2wot client will be configured with auth tokens for agents that have authorization to publish the TDs they manage. The internal rule is that the Thing ID of published Things must contain the authenticated agent ID prefix. Only agents are allowed to publish TDs.

If go2wot is used as part of a separate standalone application, then the application must manage the access control to allow agents to publish their TD based on how what directory service is configured.

## Build and Install

See [docs/INSTALL.md](docs/INSTALL.md)

## Configuration

todo
