# PacketCapture
C# code for winPcap

SDL Network Interface

	Proposal: SDL Network Interface
	Author: Varun Chari and Basvaraj Tonshal
	Status: Awaiting Review
	Impacted Platforms: SDL Core, Android, iOS

Introduction

In-Vehicle Distributed Connectivity allows the (IVDCM) allows In-vehicle applications on TCU and SYNC to receive/send data to cloud using desired networking intent. The intent can be either low level policy or high level intent. This proposal highlights how IVDCM on SYNC facilitates sending data using SYNC Wi-Fi and SDL. It allows exposing both SDL and Wi-Fi as network interfaces for the applications on SYNC, thus providing a central platform for managing connection interfaces on SYNC. This proposal highlights integration with SDL and not SYNC Wi-Fi

Motivation

Currently to upload or download, SDL enabled application on SYNC, and Mobile, utilizes application layer protocols like HTTP on the phone by sending SDL RPC messages. For example, if a SDL enabled application on SYNC needs to get some data from the cloud, the HTTP request is serialized into JSON and sent to the phone; phone then utilizes the HTTP service and makes a request to get the data. It is worth noting that, application on SYNC assumes the availability of HTTP service on the phone. Since HTTP is a basic service and is supported on all smartphones, this method works. Let’s assume that an application wants to do a MQTT request, unfortunately MQTT is not a universally supported service like HTTP and is not supported and thus we won’t be able to make the request. The primary motivation of this proposal is to make application on SYNC make IP data request through phone using SDL, thus becoming agnostic to application layer protocol support on the mobile.

Proposed solution

The solution detailed in this proposal will introduce IVDCM, which would be responsible for maintaining a single list of available network interfaces. IVDCM plugin would facilitate communication between SDL and IVDCM and also allow applications to use IVDCM APIs. Depending upon the type of data, IP or non-IP, IVDCM will decide which mechanism to follow.

The non-IP data processing is similar to using HTTP service, with only difference being application requests are serialized to Google Protocol Buffer (GPB) by IVDCM for internal data processing.  For example Demo application in Figure 1 makes Get/Upload request to IVDCM. IVDCM converts the request into SDL RPC message and serialize it into GPB data. The serialized GPB data is then read by IVDCM plugin and sent to phone using JSON. The phone then completes the Get/Upload request using HTTP service.

The main highlight of IVDCM is processing IP data requests directly from the SYNC application. To achieve this IVDCM creates a virtual interface (SDL tunnel) on SYNC. Consider a Demo application as shown in Figure 2 is running on SYNC.
 It can directly write the IP data onto the virtual interface. IVDCM plugin reads the IP data using packet sniffer (libpcap) library. The data is sent to phone as a hybrid message, which is combination of JSON and Bulk Data. IVDCM running on phone processes the message and if it is IP data then IVDCM creates an IP packet. The application then simply has to create a TCP socket to send/receive data, thus enabling the application running on SYNC to process any IP data request (HTTP, MQTT etc.).

![image](https://cloud.githubusercontent.com/assets/24734005/23311407/0b50fa42-fa84-11e6-8002-0d9ef62f9678.png)   ![image](https://cloud.githubusercontent.com/assets/24734005/23314862/a92b3f7c-fa91-11e6-887d-c3301584ec69.png)
Figure 1 : NON-IP data processing				Figure 2: IP data processing 




