# Old Guide - Do not edit - reference only
_(c) AMWA 2020, CC Attribution-NoDerivatives 4.0 International (CC BY-ND 4.0)_

## Document Overview
### Scope
This document is intended as a guide for implementers who want to add NMOS Authorization to their NMOS Nodes, where that Node is a typical broadcast device. (Implementation of NMOS Authorization in Nodes that act as controllers is NOT within the scope of this document [REF to NMOS Controller guide]).
### Structure

 - _Pre-requisites_ - what you need to know before you start your implementation.
 - _Authenticated API Calls_ - including NMOS Authorization in your Node implementation.
 - _Development Resources_ - useful tools and resources to help in your   
   implementation journey. 
- _References_ - In addition to the links to   
   articles, specifications and tutorials found within the guide, this  
   section contains other useful reading.
 
 ### Terminology
 The use of OAuth 2.0 terminology has been adopted in the IS-10 standard. OAuth 2.0 was originally developed for allowing webapps to access protected resource, where the webapp is the Client and the protected resource is the Resource Server.  However, the NMOS Node acts as both a client (when, for instance, registering with the NMOS Registry) AND as a Resource Server (when, for instance, accepting a connection request).  For this reason the Node is sometimes referred to as the Client and sometimes as the Resource Server depending on context.
 
## Pre-requisites
Before using this guide, you should be familiar with the following specification and technologies.
  
### NMOS IS-04, IS-05, IS-08
#### [IS-04 Registration & Discovery](https://specs.amwa.tv/is-04/) & [IS-05 Connection Management](https://specs.amwa.tv/is-05/)
As well as a familiarity with the standards, it is also assumed that you already have a working implementation of IS-04 and IS-05 for your NMOS Node.
#### [IS-08 Audio Channel Mapping](https://specs.amwa.tv/is-08/)
If your NMOS Node has audio capability, then as well as being familiar with the standard, it is assumed that you already have a working implementation of IS-08 for your NMOS Node.
#### [IS-10 Authorization](https://specs.amwa.tv/is-10/) 
The IS-10 Authorization specification is based on OAuth2 and is used for controlling access to IS-04, IS-05 and IS-08 APIs.  This guide will detail implementation of IS-10 in NMOS nodes (excluding controller nodes) for IS-04, IS-05 and IS-08 API calls.
  
### OAuth 2.0
Given that IS-10 is based on OAuth 2.0, a familiarity with the way OAuth 2.0 authorization works, and in particular the Client Credentials Grant flow, is required.  The authorization flows used by IS-10 are the Client Credentials Grant and the Authentication Code Flow. The latter is implemented when the NMOS node is a controller, and therefore beyond the scope of this guide.

The following four part series of articles give a good grounding in OAuth 2.0; the first three give an overview of OAuth 2.0 and JWT independent of implementation language. The fourth article is specific to Java implementations.

 - [Deep Dive Into OAuth2.0 and JWT (Part 1 Setting the
   Stage)](https://dzone.com/articles/deep-dive-to-oauth20-amp-jwt-part-1-setting-the-st
   "https://dzone.com/articles/deep-dive-to-oauth20-amp-jwt-part-1-setting-the-st")
  - [Deep Dive Into OAuth2.0 and JWT (Part 2
   OAuth2.0)](https://dzone.com/articles/deep-dive-to-oauth20-amp-jwt-part-2-oauth20)
   - [Deep Dive to OAuth2.0 and JWT (Part
   3)](https://dzone.com/articles/deep-dive-to-oauth20-amp-jwt-part-3-jwt)
   - [Deep Dive to OAuth2.0 and JWT (Part 4 JWT Use
   Case)](https://dzone.com/articles/what-is-zuul)

### JavaScript Web Tokens (JWT)
JavaScript Web Tokens is the preferred type of token used in IS-10.  A familiarity with JWT can be gained from the articles linked in the OAuth 2.0 section, or from this [introduction to JWT](https://jwt.io/introduction). 
  
## Implementing Authenticated API Calls

### Node to Authorization Server Interactions  
#### Overview
This section details the interactions between the NMOS Node and the Authorization Server. These include:

 - **Discovery of the Authorization Server**
 - **Client Registration**: Registration of the NMOS Node with the Authorization Server.
 - **Client Credentials Flow**: Request of an Access Token from the Authorization Server.
 - **Refreshing the Bearer Token**: Ensuring the token is always valid 
- **Fetch Authorization Server Public Keys**: To authenticate Access Tokens in incoming API calls.

![Node to Authorization Interactions](./images/client_credentials_flow.png)

The sequence diagram above gives an overview of these interaction which are described in more detail below.
#### Discovery of the Authorization Server
In order to interact with the Authorization Server, you need to know where it is.  IS-10 specifies that the [Authorization Server should use unicast DNS-SD](https://specs.amwa.tv/is-10/branches/v1.0-dev/docs/3.0._Discovery.html#dns-sd-advertisemen) to advertise itself to the  Node.

Once the node knows the whereabouts of the Authorization Server it can then fetch the [Authorization Server Metadata](https://specs.amwa.tv/is-10/branches/v1.0-dev/docs/3.0._Discovery.html#authorization-server-metadata-endpoint) to obtain supported features and endpoints.

Example request to get server metadata:

    GET /.well-known/oauth-authorization-server HTTP/1.1
    Host: authorization-server.com

Example server metadata response:

	HTTP/1.1 200 OK
	Content-Type: application/json

	{
	  "issuer": "https://authorization-server.com",
	  "authorization_endpoint": "authorization-server.com/authorize",
	  "token_endpoint": "https://authorization-server.com/token",
	  "token_introspection_endpoint": "https://authorization-server.com/introspect",
	  "userinfo_endpoint": "https://authorization-server.com/userinfo",
	  "end_session_endpoint": "https://authorization-server.com/logout",
	  "jwks_uri": "https://authorization-server.com/jwks",	  
	  "grant_types_supported": [
	    "authorization_code",
	    "implicit",
	    "refresh_token",
	    "password",
	    "client_credentials"
	  ],
	  "response_types_supported": [
	    "code",
	    "none",
	    "id_token",
	    "token",
	    "id_token token",
	    "code id_token",
	    "code token",
	    "code id_token token"
	  ],	  
	  "registration_endpoint": "https://authorization-server.com/register",	  _methods_supported": [
	    "private_key_jwt",
	    "client_secret_basic",
	    "client_secret_post",
	    "tls_client_auth",
	    "client_secret_jwt"
	  ],
	  th_signing_al"token_endpoint_auth
	  "scopes_supported": [
	    "openid",
	    "connection",
	    "node",
	    "query",
	    "registration"
	  ],
	  "code_challenge_methods_supported": [
	    "plain",
	    "S256"
	  ]
	}

#### Client Registration 
NMOS nodes first needs to register with the Authorization Server.  This is usually a one time operation that a Node would typically perform when first activated on the network.

The registration is done via a [dynamic client registration](https://specs.amwa.tv/is-10/branches/v1.0-dev/docs/4.2._Behaviour_-_Clients.html#client-registration) with the Authorization Server. The registration includes the expected grant type, which should be set to  [client credentials grant](https://specs.amwa.tv/is-10/branches/v1.0-dev/docs/4.2._Behaviour_-_Clients.html#client-credentials)

Depending on how the Authorization Server has been configured, it is likely that the Client Registration will need to be authenticated using an Initial Access Token.  This token will be generated by the Authentication Server, and provided to the NMOS Node by some proprietary method. 

Example client registration request including an Initial Access Token (This is the `Authorization: Bearer` in the HTTP header):

    POST /register HTTP/1.1
    Host: authorization-server.com
    Content-Type: application/json
    Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCIg...
    
	{
	 "client_name": "My Example Client",
	 "grant_types": ["client_credentials"],
	 "jwks_uri": "https://client.example.org/my_public_keys.jwks",
	 "response_types": ["none"],
	 "scope": "registration",
	 "token_endpoint_auth_method": "private_key_jwt"
	}

Example response:

    HTTP/1.1 201 Created
    Content-Type: application/json
    	
    {
     "client_id": "xxxxxxxxxx",
     "client_name":"My Example Client",
     "client_id_issued_at": 1611940142,
     "grant_types":["client_credentials"],
     "jwks_uri":"https://client.example.org/my_public_keys.jwks",
     "registration_access_token":"eyJhbGciOiJIUzI1NiIsInR5c...",
     "registration_client_uri":"https://authorization-server.com/xxxxxxxxxx",
     "response_types":[],
     "token_endpoint_auth_method":"private_key_jwt"
     }
#### Client Credentials Flow
The Node can now request a bearer token from the Authorization Server using the [client credentials flow](https://specs.amwa.tv/is-10/branches/v1.0-dev/docs/4.3._Behaviour_-_Token_Requests.html#access-token-request-and-response). This access token will allow the Node to authenticate API calls on other NMOS Nodes, for instance, the NMOS Registry.

Example request to get bearer token:

	POST /token HTTP/1.1
	Host: authorization-server.com
    Content-Type: application/x-www-form-urlencoded
    
    grant_type=client_credentials
    &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
    &client_assertion=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.ey...
    &client_id=xxxxxxxxxx
    &scope=registration

In this request the `client_assertion` is a self signed JWT and `client_id` is the result from client registration.

Example token response:

	HTTP/1.1 200 OK
	Content-Type: application/json
	
	{
	 "access_token":"eyJhbGciOiJSUzUxMiIsInR5cCIgOiAiSl...",
	 "expires_in":180,
	 "scope":"registration",
	 "token_type":"bearer"
	}

#### Refreshing the Bearer Token
The Bearer Token has a limited life specified in seconds by the value of the `expires_in` parameter.  The Node should refresh the token before [it exceeds its half life](https://specs.amwa.tv/is-10/branches/v1.0-dev/docs/4.4._Behaviour_-_Access_Tokens.html#access-token-lifetime) (if the token lifetime is 30 seconds, then it should be refreshed at least 15 seconds before token expiry). 

Unlike the Authorization Code Flow, the Client Credential Flow does not issue a refresh token 'refreshing' the token simply involves repeating the Client Credential Flow.

#### Fetch Authorization Server Public Keys
As well as acquiring a token to allow this Node to register, the Node will also need the [Authorization Server's public keys](https://specs.amwa.tv/is-10/branches/v1.0-dev/docs/4.5._Behaviour_-_Resource_Servers.html#public-keys) so that it can authenticate calls on its own APIs.  These keys should be fetched every hour to ensure the keys are always up to date.

![Node fetches Authorization Server Public Keys](./images/fetch_public_keys.png)

Example request to get server public keys:

	GET /jwks HTTP/1.1
    Host: authorization-server.com
    
Example server public keys response:

	HTTP/1.1 200 OK
	Content-Type: application/json

	[
	 {
	  "alg":"RS256",
	  "e":"AQAB",
	  "kid":"aRxkfaepyicuogJnoaXfUJAEDwiDQ3o914n2JNqToZ0",
	  "kty":"RSA",
	  "n":"jBRq3QfleVgYxjS3q-tmK8686Pc2HvR50kxfB6l...",
	  "use":"sig",
	  "x5c":["MIICmzCCAYMCBgFxiHgcuzANBgkqhkiG9w0BAQ..."],
	  "x5t":"qaEz9PpliodKhNXA5jqiUky5-RU",
	  "x5t#S256":"hfJS5jB9chso-iMQ7-QNAIXFPFwz6SjrohG81r6IxyE"
	 },
	 {
	  "alg":"ES256",
	  "crv":"P-256",
	  "kid":"jMRpEWZ8_-1pmdpGqEo4ZSCb7pltOVuoQQc46aYa7RM",
	  "kty":"EC",
	  "use":"sig",
	  "x":"e7DQRay3ZWCj-Y_-Ww-QN-m95KV2IBVpZ2raP3CF5XU",
	  "y":"ay2UR1ohsrWhJgsv8T0cV66yivD4kA9_3YV8RJFjD3k"
	 },
	 {
	  "alg":"RS512",
	  "e":"AQAB",
	  "kid":"O4QEicS70s1DWFyt84niI80Z2SLsdNrVyeGwJe8g8qw",
	  "kty":"RSA",
	  "n":"yeSbbHw18xN3hh_VeHpSI01Fcp0xaI1znmWBVkmTYa...",
	  "use":"sig",
	  "x5c":["MIIFlTCCA32gAwIBAgICEAAwDQYJKoZIhvcNAQELBQAw..."],
	  "x5t":"LD3awp-sYoVbcfwOgB9BRO4HFYQ",
	  "x5t#S256":"yCrclAKahB6SE68rbx5cRwuBZoeTXfW9smoLgt6u9t4"
	 }
	]

Note that if a Node is unable to contact an Authorization Server, it should attempt to contact another Authorization Server from the discovered list until this [either succeeds or the list is exhausted](https://specs.amwa.tv/is-10/branches/v1.0-dev/docs/4.5._Behaviour_-_Resource_Servers.html#public-keys). 
  
### Node to Registry Interactions (IS-04 Registration API)  
Once the Node has registered with the Authorization Server, and acquired a Bearer Token, the next step is for it to register with the NMOS Registry using the IS-04 API, by including the Bearer Token in the API call.

![Node to Registry Interaction](./images/node_to_registry.png)

Example resource registration request:

	POST /x-nmos/registration/resource HTTP/1.1
    Host: registry.example.org
    Content-Type: application/json
    Authorization: Bearer eyJhbGciOiJSUzUxMiIsInR5cCIgOiAiSl...
    
    {
	 "data": {
	  "api": {
	   "endpoints": [
	    {
         "authorization":true,
         "host": "client.example.org",
         "port": 443,
         "protocol": "https"
        }
       ],
	   "versions": ["v1.0","v1.1","v1.2","v1.3"]
	  },
	  "caps": {},
	  "clocks": [
	   {
	    "name": "clk0",
	    "ref_type": "internal"
	   }
	  ],
	  "description": "My Example Client",
	  "hostname": "client.example.org",
	  "href": "https://client.example.org/",
	  "id": "f6545751-b2f9-5c52-a4b8-a02c0e0c2368",
	  "interfaces": [
	   {
	    "chassis_id": "5e-21-51-e3-62-d0",
	    "name": "{0BAFC19E-7B18-4C51-A7B1-9670E0B1CFE1}",
	    "port_id": "3c-52-82-6f-c2-3e"
	   },
	   {
	    "chassis_id": "5e-21-51-e3-62-d0",
	    "name": "{210054E7-4A13-414D-837D-A4CC4811DE14}",
	    "port_id": "0a-00-27-00-00-06"
	   }
	  ],
	  "label": "My Example Client",
	  "services": [],
	  "tags": {},
	  "version": "1612793440:516299600"
	 },
	 "type": "node"
	}

Example response:

	HTTP/1.1 201 Created
	Content-Type: application/json

	 {
	  "api": {
	   "endpoints": [
	    {
         "authorization": true,
         "host": "client.example.org",
         "port": 443,
         "protocol": "https"
        }
       ],
	   "versions": ["v1.0","v1.1","v1.2","v1.3"]
	  },
	  "caps": {},
	  "clocks": [
	   {
	    "name": "clk0",
	    "ref_type": "internal"
	   }
	  ],
	  "description": "My Example Client",
	  "hostname": "client.example.org",
	  "href": "https://client.example.org/",
	  "id": "f6545751-b2f9-5c52-a4b8-a02c0e0c2368",
	  "interfaces": [
	   {
	    "chassis_id": "5e-21-51-e3-62-d0",
	    "name": "{0BAFC19E-7B18-4C51-A7B1-9670E0B1CFE1}",
	    "port_id": "3c-52-82-6f-c2-3e"
	   },
	   {
	    "chassis_id": "5e-21-51-e3-62-d0",
	    "name": "{210054E7-4A13-414D-837D-A4CC4811DE14}",
	    "port_id": "0a-00-27-00-00-06"
	   }
	  ],
	  "label": "My Example Client",
	  "services": [],
	  "tags": {},
	  "version": "1612793440:516299600"
	 },
	 "type": "node"
	}

### Controller to Node (IS-04, IS-05 and IS-08 API)  

![Controller to Registry](./images/controller_to_registry.png)

Controller can use the IS-04 Query API to obtian all the registered NMOS nodes resources via the NMOS Registry.

Example query request including Access Token (This is the `Authorization Bearer` in the HTTP header):

	GET /x-nmos/query/v1.3/nodes HTTP/1.1
    Host: registry.example.org
    Authorization: Bearer eyJhbGciOiJSUzUxMiIsInR5cCIgOiAiSl...
 
Example query response:

	HTTP/1.1 200 OK
	Content-Type: application/json
	
	[
	 {
	  "api": {
	   "endpoints": [
	    {
	     "authorization": true,
	     "host": "client.example.org",
	     "port": 443,
	     "protocol": "https"
	    }
	   ],
	   "versions": ["v1.0","v1.1","v1.2","v1.3"]
	  },
	  "caps": {},
	  "clocks": [
	   {
	    "name": "clk0",
	    "ref_type": "internal"
	   }
	  ],
	  "description": "My Example Client",
	  "hostname": "client.example.org",
	  "href": "https://client.example.org/",
	  "id": "6efd8771-f827-51da-87b4-e045935c9445",
	  "interfaces": [
	   {
	    "attached_network_device": {
	     "chassis_id": "08-00-27-e8-db-cb",
	     "port_id": "08-00-27-e8-db-cb"
	    },
	    "chassis_id": "5e-21-51-e3-62-d0",
	    "name": "{0BAFC19E-7B18-4C51-A7B1-9670E0B1CFE1}",
	    "port_id": "3c-52-82-6f-c2-3e"
	   },
	   {
	    "chassis_id": "5e-21-51-e3-62-d0",
	    "name": "{210054E7-4A13-414D-837D-A4CC4811DE14}",
	    "port_id": "0a-00-27-00-00-06"
	   }	    
	  ],
	  "label": "My Example Client",
	  "services": [],
	  "tags": {},
	  "version": "1612797431:613972700"
	 },
	 {
	  "api": {
	   "endpoints": [
	    {
	     "authorization": true,
	     "host": "registry.example.org",
	     "port": 443,
	     "protocol": "https"
	    }
	   ],
	   "versions": ["v1.0","v1.1","v1.2","v1.3"]
	  },
	  "caps": {},
	  "clocks": [],
	  "description": "My Example Registry",
	  "hostname": "registry.example.org",
	  "href": "https://registry.example.org/",
	  "id": "a7bd10e7-71e1-5955-a2b0-c80461ebaed0",	  
	  "interfaces": [	   
       {
        "chassis_id": "5e-21-51-e3-62-d0",
        "name": "{4F56D5AA-AAE3-44AC-94AC-6F1E6B555F66}",
        "port_id": "0a-00-27-00-00-08"
       },
       {
        "chassis_id": "5e-21-51-e3-62-d0",
        "name": "{621847D4-708F-44EA-8F2C-828190A923BC}",
        "port_id": "02-00-4c-4f-4f-50"
       }      
      ],
      "label": "My Example Registry",
      "services": [
       {
        "href": "https://registry.example.org/x-dns-sd/v1.0",
        "type": "urn:x-dns-sd/v1.0"
       },
       {
        "href": "https://registry.example.org/x-dns-sd/v1.1",
        "type": "urn:x-dns-sd/v1.1"
       }
      ],
      "tags": {},
      "version": "1612796262:305700600"
     }
	]

After Controller has obtained all resources, it can then be using IS-05 Connection API to extract sender SDP to make connection between receiver and sender.

![Controller to Registry](./images/controller_to_nodes.png)

Exmaple using node request to get sender SDP including Access Token (This is the `Authorization Bearer` in the HTTP header):

	GET /x-nmos/connection/v1.1/single/senders/{senderId}/transportfile HTTP/1.1
	Host: client.example.org
    Authorization: Bearer eyJhbGciOiJSUzUxMiIsInR5cCIgOiAiSl...

Example sender SDP response:

	HTTP/1.1 200 OK
	Content-Type: application/sdp

	v=0
	o=- 3821785015 3821785015 IN IP4 10.1.0.29
	s=Sony Windows Node/sender/v0
	t=0 0
	a=group:DUP PRIMARY SECONDARY
	m=video 5004 RTP/AVP 96
	c=IN IP4 239.255.255.0/32
	a=ts-refclk:localmac=5E-21-51-E3-62-D0
	a=mediaclk:direct=0
	a=source-filter: incl IN IP4 239.255.255.0 10.1.0.29
	a=rtpmap:96 raw/90000
	a=fmtp:96 width=1920; height=1080; exactframerate=25; interlace; sampling=YCbCr-4:2:2; depth=10; colorimetry=BT709; TCS=SDR; PM=2110GPM; SSN=ST2110-20:2017; TP=2110TPN; 
	a=mid:PRIMARY
	m=video 5004 RTP/AVP 96
	c=IN IP4 239.255.255.1/32
	a=ts-refclk:localmac=0A-00-27-00-00-0F
	a=mediaclk:direct=0
	a=source-filter: incl IN IP4 239.255.255.1 169.254.66.179
	a=rtpmap:96 raw/90000
	a=fmtp:96 width=1920; height=1080; exactframerate=25; interlace; sampling=YCbCr-4:2:2; depth=10; colorimetry=BT709; TCS=SDR; PM=2110GPM; SSN=ST2110-20:2017; TP=2110TPN; 
	a=mid:SECONDARY

Example using connection request including Access Token (This is the `Authorization Bearer` in the HTTP header):

	PATCH /x-nmos/connection/v1.1/single/receivers/{receiverId}/staged HTTP/1.1
    Host: client.example.org
    Content-Type: application/json
    Authorization: Bearer eyJhbGciOiJSUzUxMiIsInR5cCIgOiAiSl...
    
	{
	 "transport_params": [
	  {
	   "destination_port": 5004,
	   "multicast_ip": "239.255.255.0",
	   "source_ip": "10.1.0.29",
	   "rtp_enabled": true
	  },
	  {
	   "destination_port": 5004,
	   "multicast_ip": "239.255.255.1",
	   "source_ip": "169.254.66.179",
	   "rtp_enabled": true
	  }
	 ],
	 "activation": {
	   "mode": "activate_immediate"
	 },
	 "master_enable": true,
	 "sender_id": "8d26d68a-93f2-55ff-9495-acbb576897fd",
	 "transport_file": {
	   "data": "v=0\r\no=- 3821785015 3821785015 IN IP4 10.1.0.29\r\ns=Sony Windows Node/sender/v0\r\nt=0 0\r\na=group:DUP PRIMARY SECONDARY\r\nm=video 5004 RTP/AVP 96\r\nc=IN IP4 239.255.255.0/32\r\na=ts-refclk:localmac=5E-21-51-E3-62-D0\r\na=mediaclk:direct=0\r\na=source-filter: incl IN IP4 239.255.255.0 10.1.0.29\r\na=rtpmap:96 raw/90000\r\na=fmtp:96 width=1920; height=1080; exactframerate=25; interlace; sampling=YCbCr-4:2:2; depth=10; colorimetry=BT709; TCS=SDR; PM=2110GPM; SSN=ST2110-20:2017; TP=2110TPN; \r\na=mid:PRIMARY\r\nm=video 5004 RTP/AVP 96\r\nc=IN IP4 239.255.255.1/32\r\na=ts-refclk:localmac=0A-00-27-00-00-0F\r\na=mediaclk:direct=0\r\na=source-filter: incl IN IP4 239.255.255.1 169.254.66.179\r\na=rtpmap:96 raw/90000\r\na=fmtp:96 width=1920; height=1080; exactframerate=25; interlace; sampling=YCbCr-4:2:2; depth=10; colorimetry=BT709; TCS=SDR; PM=2110GPM; SSN=ST2110-20:2017; TP=2110TPN; \r\na=mid:SECONDARY\r\n",
	   "type": "application/sdp"
	  }
	}

Example connection response:

	HTTP/1.1 200 OK
	Content-Type: application/json

	{
	 "activation": {
	   "activation_time": "1612804428:810443300",
	   "mode": "activate_immediate",
	   "requested_time": null
	  },
	 "master_enable": true,
	 "sender_id": "8d26d68a-93f2-55ff-9495-acbb576897fd",
	 "transport_file": {
	   "data": "v=0\r\no=- 3821785015 3821785015 IN IP4 10.1.0.29\r\ns=Sony Windows Node/sender/v0\r\nt=0 0\r\na=group:DUP PRIMARY SECONDARY\r\nm=video 5004 RTP/AVP 96\r\nc=IN IP4 239.255.255.0/32\r\na=ts-refclk:localmac=5E-21-51-E3-62-D0\r\na=mediaclk:direct=0\r\na=source-filter: incl IN IP4 239.255.255.0 10.1.0.29\r\na=rtpmap:96 raw/90000\r\na=fmtp:96 width=1920; height=1080; exactframerate=25; interlace; sampling=YCbCr-4:2:2; depth=10; colorimetry=BT709; TCS=SDR; PM=2110GPM; SSN=ST2110-20:2017; TP=2110TPN; \r\na=mid:PRIMARY\r\nm=video 5004 RTP/AVP 96\r\nc=IN IP4 239.255.255.1/32\r\na=ts-refclk:localmac=0A-00-27-00-00-0F\r\na=mediaclk:direct=0\r\na=source-filter: incl IN IP4 239.255.255.1 169.254.66.179\r\na=rtpmap:96 raw/90000\r\na=fmtp:96 width=1920; height=1080; exactframerate=25; interlace; sampling=YCbCr-4:2:2; depth=10; colorimetry=BT709; TCS=SDR; PM=2110GPM; SSN=ST2110-20:2017; TP=2110TPN; \r\na=mid:SECONDARY\r\n",
	   "type": "application/sdp"
	  },
	 "transport_params": [
	  {
	   "destination_port": 5004,
	   "interface_ip": "auto",
	   "multicast_ip": "239.255.255.0",
	   "rtp_enabled": true,
	   "source_ip": "10.1.0.29"
	  },
	  {
	   "destination_port": 5004,
	   "interface_ip": "auto",
	   "multicast_ip": "239.255.255.1",
	   "rtp_enabled": true,
	   "source_ip": "169.254.66.179"
	  }
	 ]
	}

Description of validation of token by comparison to public key of authentication server.

5) Controller sends access token to Node as part of HTTP headers on access to resources
6) Node has to verify the access token:
6a) Verify the signature of the token. This requires obtaining the public key from the authorisation server, and then applying a "RSASSA-PKCS1-v1_5 using SHA-512" check on the signature in the token
6b) Verify the fields of the token are correct, including expiry time, iat/nbf (IS-10 doesn't mention nbf, but other JWT texts do), aud and iss, plus x-nmos-* claims.
7) If stage 6 passes, then return the requested data, otherwise return an authorisation failure HTTP code.

_PB: as required, further guidance on spec details e.g. the NBF field question_  

### Event & Tally (IS-07)
_MF: I believe the pattern for this API is different, as it can include node-to-node communication, and is mainly based around websockets or MQTT, rather than traditional HTTP requests_

[ Sequence Diagram - Simon TODO]  

[Simon to provide some examples]
Example - initiating web socket connection - decorate with token
Example - accepting web socket connection - validate the token

Within current specification, websocket can be secured with a token, MQTT not security no currently secured by 

Websocket is secured, not the API calls. 

## Development Resources 
### Authorization Server
[An auth server is required - link to open source implementations such as Keycloak]
_MF: Guidance on how to set it up for this. See AB wiki guide plus how to segment network. AB: maybe BBC could provide container with config, helped with docker-compose._  
  
### NMOS Testing Tool
[Description and links to the NMOS Testing tool]  
  
### VPN-based testing and virtual workshops
_[Question Only available to AMWA members, so probably best excluded from this guide?]_

_PB: yes, but could reference how we did this (TE blog post++)_
### JavaScript Web Token Tools
This [online tool](https://jwt.io/#debugger-io) is a useful webapp for experimenting with building and decoding JWTs.


## References
-   external OAuth 2.0 and JWT tutorials
-   Sony slides (updated)
-   not-as-yet-written IS-04/05/08 implementers' guides
-   Existing wiki implementers info
-   Initial white papers from BBC










<!--stackedit_data:
eyJkaXNjdXNzaW9ucyI6eyJ1ektVZ3Nwa3pkQnZxOFBKIjp7In
RleHQiOiIxKSBDb250cm9sbGVyIGlzIHJlZ2lzdGVyZWQgd2l0
aCB0aGUgYXV0aG9yaXNhdGlvbiBzZXJ2ZXIgKGRvZXMgdGhhdC
BhcHBseSwgYXPigKYiLCJzdGFydCI6MjI1OTcsImVuZCI6MjI1
OTd9LCJOYWxYdWRkc1ZBMDRNWVBQIjp7InRleHQiOiJFdmVudC
AmIFRhbGx5IChJUy0wNykiLCJzdGFydCI6MjMyODAsImVuZCI6
MjMzMDF9LCJlUGdHREF0OHB2QWRuZW9mIjp7InRleHQiOiIjIy
MjIFtJUy0wNCBSZWdpc3RyYXRpb24gJiBEaXNjb3ZlcnldKGh0
dHBzOi8vc3BlY3MuYW13YS50di9pcy0wNC8pICYgW0lTLTA1IE
Nv4oCmIiwic3RhcnQiOjE2MjYsImVuZCI6MTcyNX0sImgxbDFO
aEpabWtFZWtlWjIiOnsidGV4dCI6IiMjIyBOTU9TIElTLTA0LC
BJUy0wNSwgSVMtMDgiLCJzdGFydCI6MTU5NywiZW5kIjoxNjI1
fSwiQlROYlFFZmt1bVJVRW56aCI6eyJ0ZXh0IjoiTm9kZSB0by
BSZWdpc3RyeSBJbnRlcmFjdGlvbnMgKElTLTA0IFJlZ2lzdHJh
dGlvbiBBUEkpIiwic3RhcnQiOjEyMzc3LCJlbmQiOjEyNDMxfS
wicW1VanpuR0txUEphamRhdiI6eyJ0ZXh0IjoiVGhlIHVzZSBv
ZiBPQXV0aCAyLjAgdGVybWlub2xvZ3kgaGFzIGJlZW4gYWRvcH
RlZCBpbiB0aGUgSVMtMTAgc3RhbmRhcmQuIE9BdXRo4oCmIiwi
c3RhcnQiOjkyOCwiZW5kIjoxNDc0fSwiNjFHVE1xWDg1dmRLVU
tlVCI6eyJ0ZXh0IjoiQ29udHJvbGxlciB0byBOb2RlIiwic3Rh
cnQiOjE0OTEwLCJlbmQiOjE0OTI4fX0sImNvbW1lbnRzIjp7Im
twNVFDYXVGNGZ3YkdZYmwiOnsiZGlzY3Vzc2lvbklkIjoidXpL
VWdzcGt6ZEJ2cThQSiIsInN1YiI6ImdoOjMwMjMxMzIxIiwidG
V4dCI6Im1heWJlIHRoaXMgYml0IHNob3VsZCBiZSBkZXNjcmli
ZWQgaW4gYW5vdGhlciBkb2MgIENvbnRyb2xsZXIgSW1wbGVtZW
5hdGVyIEd1aWRlIHNlY3Rpb24gQ29udHJvbGxlciB0byBBdXRo
ZW50aWNhdGlvbiBTZXJ2ZXIgSW50ZXJhY3Rpb25zIiwiY3JlYX
RlZCI6MTYxMjM5ODg1NDQ1OX0sImV0VThmeU45Y09Ld3BLbUIi
OnsiZGlzY3Vzc2lvbklkIjoiTmFsWHVkZHNWQTA0TVlQUCIsIn
N1YiI6ImdoOjY0NDEwMTE5IiwidGV4dCI6IklzIHRoaXMgYmV5
b25kIHRoZSBzY29wZSBvZiB0aGlzIGd1aWRlPyIsImNyZWF0ZW
QiOjE2MTI0NTk2MzAwNjZ9LCJvUnVXck5BZFZCZmp4SkxKIjp7
ImRpc2N1c3Npb25JZCI6ImVQZ0dEQXQ4cHZBZG5lb2YiLCJzdW
IiOiJnaDo2NDQxMDExOSIsInRleHQiOiJMaW5raW5nIHRvIHJl
c291cmNlcyBmcm9tIHRpdGxlIFtjb21wYWN0XSwgb3IgYXMgZX
hwbGljaXQgbGluayBpbiB0aGUgcHJvc2UgW3VzZWZ1bCBmb3Ig
bXVsdGlwbGUgcmVzb3VyY2VzXT8gT3IgbWl4ZWQvYm90aD8iLC
JjcmVhdGVkIjoxNjEyNTE4MTI1MDAwfSwiSEkyRUt5SUQxUmFH
eVk3USI6eyJkaXNjdXNzaW9uSWQiOiJoMWwxTmhKWm1rRWVrZV
oyIiwic3ViIjoiZ2g6NjQ0MTAxMTkiLCJ0ZXh0IjoiSVMtMDc/
Pz8/PyIsImNyZWF0ZWQiOjE2MTI1Mzc1NDM5ODJ9LCJGQUFXNT
B6U2FZSHlkbkt6Ijp7ImRpc2N1c3Npb25JZCI6IkJUTmJRRWZr
dW1SVUVuemgiLCJzdWIiOiJnaDo2NDQxMDExOSIsInRleHQiOi
JNRiBXcm90ZTpcbjUuIERvZXMgYSBkZXNjcmlwdGlvbiBvZiB0
aGUgcmVnaXN0cnkgYmVoYXZpb3VyIG9uIHJlY2VpcHQgb2YgdG
hlIGFjY2VzcyB0b2tlbiBtYWtlIHNlbnNlIGhlcmUsIG9yIHNo
b3VsZCB0aGF0IGdvIGVsc2V3aGVyZSBhbmQgaGF2ZSBhIGxpbm
sgYmFjayB0byBoZXJlPyBJIGJlbGlldmUgaXQgaXMgZXNzZW50
aWFsbHkgdGhlIHNhbWUgdGhpbmcgYXMgc3RhZ2UgNiBvZiB0aG
UgXCJjb250cm9sbGVyIHRvIG5vZGVcIiBpbnRlcmFjdGlvbiBi
ZWxvdy4gIFxuNi4gSWYgSSB1bmRlcnN0YW5kIGNvcnJlY3RseS
wgdGhlIHJlZnJlc2ggdG9rZW4gZG9lcyBub3QgYXBwbHkgKG9y
IGRvZXNuJ3QgbWFrZSBtdWNoIHNlbnNlKSBpbiB0aGUgY2xpZW
50IGNyZWRlbnRpYWxzIGdyYW50IHdvcmtmbG93LiBTbyB3aGVu
IHRoZSBhY2Nlc3MgdG9rZW4gZXhwaXJlcyAoY2hlY2tpbmcgSU
FUIGl0c2VsZiwgb3IgYXV0aG9yaXNhdGlvbiBmYWlsdXJlIHJl
dHVybiBmcm9tIHJlZ2lzdHJ5KSwgdGhlIG5vZGUganVzdCByZX
BlYXRzIHN0ZXAgKDIpIiwiY3JlYXRlZCI6MTYxMjU0MDY0NTM4
Mn0sImtyOVQ4WkVnV3I5SktNRUYiOnsiZGlzY3Vzc2lvbklkIj
oiQlROYlFFZmt1bVJVRW56aCIsInN1YiI6ImdoOjY0NDEwMTE5
IiwidGV4dCI6IlNpbW9uIHJlc3BvbmRlZDpcbmRlc2NyaWJlZC
BpbiBzdGVwIDUgb2YgTm9kZSB0byBBdXRoZW50aWNhdGlvbiBT
ZXJ2ZXIgSW50ZXJhY3Rpb25zIiwiY3JlYXRlZCI6MTYxMjU0MD
Y2NDI5N30sInlkbmJEUWZUYWpLSFEwd2wiOnsiZGlzY3Vzc2lv
bklkIjoicW1VanpuR0txUEphamRhdiIsInN1YiI6ImdoOjY0ND
EwMTE5IiwidGV4dCI6IklzIHRoaXMgYWNjdXJhdGU/IEFuZCBy
ZWFzb25hYmxlPyIsImNyZWF0ZWQiOjE2MTI1NDEwOTkzMzR9LC
JsbU9jdXVtMlZySFo4WFc4Ijp7ImRpc2N1c3Npb25JZCI6IjYx
R1RNcVg4NXZkS1VLZVQiLCJzdWIiOiJnaDo2NDQxMDExOSIsIn
RleHQiOiJEbyB3ZSBqdXN0IGNvdmVyIHZhbGlkYXRpb24gb2Yg
QVBJIGNhbGxzPyIsImNyZWF0ZWQiOjE2MTI1NDE2MzAzMDB9LC
JvWnMzdUU5eVZFQWJZdG9NIjp7ImRpc2N1c3Npb25JZCI6IjYx
R1RNcVg4NXZkS1VLZVQiLCJzdWIiOiJnaDo2NDQxMDExOSIsIn
RleHQiOiJpLmUuIGFsbCB0aGUgaW5jb21pbmcgQVBJIGNhbGxz
IGZyb20gdGhlIENvbnRyb2xsZXI/IiwiY3JlYXRlZCI6MTYxMj
U0MTkxMTY4OX19LCJoaXN0b3J5IjpbLTM2OTg4NjU5N119
-->