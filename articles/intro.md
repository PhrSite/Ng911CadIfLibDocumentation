# Using the NG911CadIfLib Class Library

The [Ng911CadIfServer](~/api/Ng911CadIfLib.Ng911CadIfServer.yml) class performs the following roles.
1. Implements the NENA EIDO Conveyance Protocol defined in [NENA-STA-024.1a-2023](https://cdn.ymaws.com/www.nena.org/resource/resmgr/standards/nena-sta-024.1a-2023_eidocon.pdf). This protocol uses Web Sockets as the transport layer.
2. Manages EIDO subscriptions from multiple CAD systems or PSAP CHFEs from external agencies that wish to receive EIDOs
3. Provides the EIDO Retrieval Service specified in NENA-STA-024-1a-2023. See the [EIDO Retrieval Service](#retrievalservice).
4. Notifies the application when a new subscription request is received or a subscription is terminated
4. Provides an interface to one ore more NG9-1-1 Event Loggers and logs subscribe/notify and EIDO delivery events. See [NG9-1-1 Event Logging](#eventlogging)

An application that uses the Ng911CadIfServer class is reponsible for the following.
1. Create an EIDO when an NG9-1-1 call is received
2. Send the EIDO to all subscribed CAD or remote agencies using an API call to the Ng911CadIfServer class
3. Store the EIDOs for all active incidents/calls
4. Send all EIDOs for active incidents when a new subscription request is received
5. Respond to events exposed by the Ng911CadIfServer class
6. Authenticate clients when they connect to the Ng911CadIfServer object. See [Mutual Authentication](#mutualauthentication).


## Project Preparation
Follow the instructions in this section to add the Ng911CadIfLib class library to a PSAP CHFE project.

Add the following framework reference to the application's *.csproj file if it is not already included.
```
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>
```

Add the following NuGet packages to the application.
1. Ng911CadIfLib
2. Ng911Lib
3. EidoLib
4. NewtonSoft.Json

The [Ng911Lib](https://phrsite.github.io/Ng911LibDocumentation/) NuGet package is a class library that contains classes for all of the JSON and XML schemas used in NG9-1-1 applications, except for the Emergency Incident Data Object (EIDO).

The [EidoLib](https://phrsite.github.io/EidoLibDocumentation/) NuGet package is a class library that contains classes for all of the JSON schemas used for an EIDO. This includes all of the NIEM type definitions.

## Getting Started
Perform the following steps to use the Ng911CadIfServer class.
1. Create an instance of the Ng911CadIfServer class and pass the configuration parameters in the constructor.
2. Hook the desired events of the Ng911CadIfServer class.
3. Call the Start() method of the Ng911CadIfServer class.

When the application shuts down, it should call the Shutdown() method of the Ng911CadIfServer object. This method performs an orderly shutdown by terminating the subscription of any subscribed CAD system elements and performing a normal close for the Web Socket connections.

# Ng911CadIfServer Events
The Ng911CadIfServer class provides two events that an application can hook.
## The NewSubscription Event
The Ng911CadIfServer class fires this event when it accepts a new subscription. The application must hook this event and respond to it. The delegate type for the handler of this event is [NewSubscriptionDelegate](~/api/Ng911CadIfLib.NewSubscriptionDelegate.yml).

The event handler for this event must call the SendEidosToSubscriber() method of the Ng911CadIfServer class with the subscribtion ID provided to the application in the event handler and a list of EIDOs for active incidents. The application does not need to call the SendEidosToSubscriber() method if there are no active incidents.

The purpose of this event and the SendEidosToSubscriber() method is to meet the requirement specified in NENA-STA-024.1a-2023 that an EIDO notifier (i.e. the PSAP CHFE) send EIDOs for all currently active incidents to a new subscriber.

The Ng911CadIfServer class manages subscription state so it is not necessary for the application to manage the state of the subscription identified by the NewSubscription event.

## The SubscriptionEnded Event
The Ng911CadIfServer class fires this event when it detects that a subscription has ended. A subscription will end when one of the following events occur.
1. The subscriber ended the subscription by sending an unsubscribe message to the server
2. The subscription expired
3. The subscriber closed its end of the Web Socket connection to the server
4. The subscriber failed to respond to a event notify message from the server three times in succession

This event is for information purposes only. The application need not perform any action in response to it.

# Sending EIDOs
The PSAP CHFE application can send an EIDO to all subscribers by calling the SendEido() method of the Ng911CadIfServer class. The SendEido() method takes an [EidoType](https://phrsite.github.io/EidoLibDocumentation/api/Eido.EidoType.html) object as a parameter. The Ng911CadIfServer object will send the EIDO to each subscriber individually and wait (in a separate thread) for a response from the subscriber. This means that the SendEido() method does not block. If a subscriber is subscribed to a single incident ID, then the SendEido() method will send the EIDO to that subscriber only if the incident ID in the EIDO matches the incident ID that the subscriber subscribed to.

The PSAP CHFE system should send updated EIDO objects to the Ng911CadIfServer object when the following events occur.
1.	Updated call location information is received
2.	Updated additional call related data is received
3.	The call taker adds information to the call such as call taker notes or the call taker changes the incident type information
4.	The call taker creates a conference, adds a party to the conference or transfers the call
5.	The call is terminated

When a the Ng911CadIfServer accepts a new subscription request, it fires the NewSubscription event. The NewSubscription event handler has a string parameter called strSubscriptionId. In the event hanlder for the NewSubscription event, the PSAP CHFE can send the EIDOs for all active incidents/calls by calling the SendEidosToSubscriber() method of the Ng911CadIfServer class. The SendEidosToSubscriber() method takes the subscription ID and a list of EIDO objects.

# <a name="retrievalservice">The EIDO Retrieval Service</a>
The EIDO Retrieval Service is an HTTPS/ReSTful interface that an external agency can use to request the EIDO for a specific incident. The need for this interface arises when the PSAP CHFE application transfers a NG9-1-1 call to the external agency.

To use the EIDO retrieval service of the Ng911CadIfServer, the application configures the HTTPS base request path that the Ng911CadIfServer will listen on in the constructor of the Ng911CadIfServer object. For example: "/incidents/eido".

When the PSAP CPE application transfers a call to another agency's PSAP CHFE or invites that agency to a conference, it must construct an HTTPS URI that points to the EIDO retrieval service interface of the Ng911CadIfServer object. The request URI contains the host address of the Ng911CadIfServer object, the base request path plus a reference ID that identifies the EIDO as the last path element. For example: https://192.168.1.12:10000/incidents/eido/12345. In this example, the reference ID is "12345".

The URI for the incident's EIDO gets included in the SIP INVITE request that gets sent to the external agency's PSAP CHFE. Section 4.7.4 of [NENA-STA-010.3b-2021](https://cdn.ymaws.com/www.nena.org/resource/resmgr/standards/nena-sta-010.3b-2021_i3_stan.pdf) describes how this happens.

When the external agency's PSAP CHFE performs an HTTPS GET to the EIDO retrieval service interface of the Ng911CadIfServer object, the Ng911CadIfServer extracts the EIDO reference ID from the last path element of the request path and calls the [EidoRetrievalCallbackDelegate](~/api/Ng911CadIfLib.EidoRetrievalCallbackDelegate.yml) that the application provided in its constructor. The application then uses the reference ID parameter to get the EidoType object and return it to the Ng911CadIfServer object. The Ng911CadIfServer object then returns the EIDO in its response to the HTTPS GET request.

When the external agency's PSAP CHFE receives the EIDO, it extracts the incident ID and it may subscribe to receive updates of the EIDO for that single incident.

# <a name="eventlogging">NG9-1-1 Event Logging</a>
The Ng911CadIfServer class can support logging for NG9-1-1 Log Events. The application must perform the following steps to use this capability.
1. Create and configure an [I3LogEventClientMgr](https://phrsite.github.io/Ng911LibDocumentation/api/I3V3.LoggingHelpers.I3LogEventClientMgr.html) object
2. Create an [CadIfLoggingSettings](~/api/Ng911CadIfLib.CadIfLoggingSettings.yml) object
3. Pass the I3LogEventClientMgr and the CadIfLoggingSettings objects in the constructor of the Ng911CadIfServer class.

See [NG9-1-1 Event Logging](https://phrsite.github.io/Ng911LibDocumentation/articles/Ng911EventLogging.html).

If the application already uses the I3LogEventClientMgr class for logging NG9-1-1 events, then it can use that object in the constructor of the Ng911CadIfServer class.

## Events That Are Logged
The Ng911CadIfServer class logs the following events defined in NENA-STA-024.1a-2023.
1. EidoLogEvent
2. EidoTransmissionErrorLogEvent
3. SubscriptionRequestedLogEvent
4. SubscriptionRequestedResponseLogEvent
5. SubscriptionTerminatedLogEvent
6. SubscriptionTerminatedResponseLogEvent
7. WebSocketEstablishedLogEvent
8. WebSocketTerminatedLogEvent


# <a name="mutualauthentication">Mutual Authentication</a>
The Ng911CadIfServer has the ability to perform Ng9-1-1 mutual authentication of clients.

One of the parameters of the constructor of the Ng911CadIfServer class is a callback delegate called MutualAuthCallback. The delegate type for this parameter is [MutualAuthenticationDelegate](~/api/Ng911CadIfLib.MutualAuthenticationDelegate.yml).

The MutualAuthenticationDelegate type takes the following three parameters and returns a bool value.
1. X509Certificate2 certificate
2. X509Chain chain
3. SslPolicyErrors errors

If the application provides a MutualAuthenticationDelegate in the constructor of the Ng911CadIfServer class, then the client must provide an X509 certificate in the HTTPS connection request. The Ng911CadIfServer class will call the application provided callback function when a client attempts to connect to either the Web Sockets interface or the EIDO Retrieval Service interface. The application provided callback can return true to accept the HTTPS connection request or false to deny the connection request.

If the application does not provide a MutualAuthenticationDelegate parameter (i.e., the MutualAuthCallback constructor parameter is null) then the Ng911CadIfServer class will accept all HTTPS connection requests, regardless of whether the client provides a X.509 certificate in the connection request.

The application can use the information in the certificate, chain and errors parameters of its MutualAuthenticatationDelegate function to determine whether to accept or to reject the client's connection request.

To perform strict client certificate validation, the application can return false if the errors parameter is not equal to SslPolicyErrors.None. If the application wishes not to be strict (for example, a valid certificate that has expired is acceptable), then it can examine the information in the certificate and chain parameters to determine what to do.

If the certificate was issued by the NENA PSAP Credentialing Agency (PCA) then the application can examine the information in the Subject Alternate Name (SAN) of the certificate in order to determine what to do.

The [CertUtils](https://phrsite.github.io/Ng911LibDocumentation/api/Ng911CertUtils.CertUtils.html) class in the Ng911Lib class library contains a method called GetOtherNameParams() that can extract the following information from the client's X.509 certificate.
1. ID Type (identifies the entity such as: Element, Service, Agent or Agency)
2. ID of the entity
3. Roles assigned to the entity
4. Owner of the certificate assigned to the entity

See [Ng9-1-1 X.509 Certificates](https://phrsite.github.io/Ng911LibDocumentation/articles/Ng911Certificates.html).


