#SDL Network Interface
* Proposal: SDL Network Interface
* Authors: Varun Chari & Basavaraj Tonshal
* Status: Awaiting Review
* Impacted Platforms: SDL Core, Android, iOS


##Introduction

In-Vehicle Distributed Connectivity (IVDCM) allows In-vehicle applications on Telematics Control Unit (TCU) and In-Vehicle Infotainment (IVI) to receive/send data to cloud using desired networking intent. The intent can be either low level policy or high level intent. This proposal highlights how IVDCM on IVI facilitates sending data using IVI Wi-Fi and SDL. It allows exposing both SDL and Wi-Fi as network interfaces for the applications on IVI, thus providing a central platform for managing connection interfaces on IVI. This proposal highlights integration with SDL and not IVI Wi-Fi

##Motivation

Currently to upload or download, SDL enabled application on IVI, utilizes application layer protocols like HTTP on the phone by sending SDL RPC messages. For example, if a SDL enabled application on IVI needs to get some data from the cloud, the HTTP request is serialized into JSON and sent to the phone; phone then utilizes the HTTP service and makes a request to get the data. It is worth noting that, application on IVI assumes the availability of HTTP service on the phone. Since HTTP is a basic service and is supported on all smartphones, this method works. Let’s assume that an application wants to do a MQTT request, unfortunately MQTT is not a universally supported service like HTTP and is not supported and thus we won’t be able to make the request. The primary motivation of this proposal is to make application on IVI make IP data request through phone using SDL, thus becoming agnostic to application layer protocol support on the mobile.

##Proposed solution

The solution detailed in this proposal will introduce IVDCM, which would be responsible for maintaining a single list of available network interfaces. IVDCM plugin would facilitate communication between SDL and IVDCM and also allow applications to use IVDCM APIs. Depending upon the type of data, IP or non-IP, IVDCM will decide which mechanism to follow.

The non-IP data processing is similar to using HTTP service, with only difference being application requests are serialized to Google Protocol Buffer (GPB) by IVDCM for internal data processing.  For example Demo application in Figure 1 makes Get/Upload request to IVDCM. IVDCM converts the request into SDL RPC message and serialize it into GPB data. The serialized GPB data is then read by IVDCM plugin and sent to phone using JSON. The phone then completes the Get/Upload request using HTTP service.

The main highlight of IVDCM is processing IP data requests directly from the application on IVI. To achieve this IVDCM creates a virtual interface (SDL tunnel) on IVI. Consider a Demo application as shown in Figure 2 is running on IVI. It can directly write the IP data onto the virtual interface. IVDCM plugin reads the IP data using packet sniffer (libpcap) library. The data is sent to phone as a hybrid message, which is combination of JSON and Bulk Data. IVDCM running on phone processes the message and if it is IP data then IVDCM creates an IP packet. The application then simply has to create a TCP socket to send/receive data, thus enabling the application running on IVI to process any IP data request (HTTP, MQTT etc.). 

<img src="https://cloud.githubusercontent.com/assets/24734005/23311407/0b50fa42-fa84-11e6-8002-0d9ef62f9678.png" width="45%"></img>       <img src="https://cloud.githubusercontent.com/assets/24734005/23314862/a92b3f7c-fa91-11e6-887d-c3301584ec69.png" width="45%"></img> 

<code>Figure 1: NON-IP data processing&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Figure 2: IP data processing</code>

##Detailed Design

####•	IVDCM will provide IVDCM library for applications to use. Figure 3 shows the sequence diagram for IVDCM SDL start. On initiating IVDCM, it starts a GPB server and SDL virtual network server. GPB server is responsible for handling non-IP data whereas SDL virtual network server is responsible for handling IP data.

<img src="https://cloud.githubusercontent.com/assets/24734005/23314868/af899e72-fa91-11e6-9cba-4b414e0b8f8c.png" width="90%"></img> 
<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Figure3: Graphical Diagram of IVDCM SDL start</code>

#####•	IVDCM SDL Integration

Currently IVDCM SDL plugin supports two types of the communication with IVDCM and SDL enabled mobile app. Following protocol is used for communication between mobile and SDL. Each frame header is 12 bytes long. It is described as follows

<img src="https://cloud.githubusercontent.com/assets/24734005/23314878/b71d8536-fa91-11e6-8faf-3b01d9a15a5e.png" width="90%"></img> 
<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Figure 4: SDL Frame Header</code>


####•	IP data handling.

This is implemented using TUN device driver. TUN/TAP provides possibility to receive and transmit IP packets from user space programs. In order to use the driver a program has to open /dev/net/tun and issue a corresponding ioctl() to register a network device with the kernel.  A network device will appear as tunXX. We create sdltun0 on IVI that represents IVI. When the program closes the file descriptor, the network device and all corresponding routes will disappear. IVDCM SDL plugin receives IP packets from /dev/net/tun device, send them to the mobile application, receives response (using FORD mobile protocol) and writes them to the /dev/net/tun device. Figures 4 shows sequence diagrams for the example when a Demo application is making a Get/upload request for IP data to IVDCM and how IVDCM forwards this request to phone. Figure 5 shows the sequence of how the phone uses IVDCM to process this IP data request.

<img src="https://cloud.githubusercontent.com/assets/24734005/23314887/c72eb83c-fa91-11e6-8577-af1f70288121.png" width="90%"></img> 
<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Figure5: Graphical Diagram of Get/upload using SDL IP</code>

<img src="https://cloud.githubusercontent.com/assets/24734005/23314898/cfebfcfa-fa91-11e6-9413-b8241d9b8db6.png" width="90%"></img> 
<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Figure 6: Graphical Diagram of Mobile Application IP data processing</code>

####•	Non-IP data handling. 

It uses GPB protocol for communication with IVDCM. In this case IVDCM SDL acts as server and IVDCM as client. Communication is done using socket and GPB protocol. IVDCM SDL will be responsible to notify client regarding mobile app internet state and to handle GPB requests for download or upload using FORD mobile protocol. Figure 6 shows the sequence diagram for the example when Demo application Get/upload request for non-IP data to IVDCM and how IVDCM forwards this request to phone. Figure 7 shows the sequence of how the phone uses IVDCM to process this non-IP data request.

<img src="https://cloud.githubusercontent.com/assets/24734005/23314907/d906a7d6-fa91-11e6-8237-5f72fb0e9a7d.png" width="90%"></img> 
<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Figure 7: Graphical Diagram of Get/upload using SDL GPB</code>

<img src="https://cloud.githubusercontent.com/assets/24734005/23314921/e09623dc-fa91-11e6-83c2-3f28a3506f2d.png" width="90%"></img> 
<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Figure 8: Graphical Diagram of Mobile Application non IP data processing</code>

####•	Application on IVI:

IVDCM provides notification regarding the status of the network interfaces to the application. The application can subscribe to these notifications. The following application shows how the application can subscribe to notifications and also IP and non-IP data handling using IVDCM APIs.

```c++
#include <unistd.h>
#include <stdio.h>
#include <pthread.h>
#include <iostream>
#include <string>

#include "dcml_lib_interface/dcml.h"
#include <dcm_lib/connectivity_manager_library_listener.h>
#include <dcm_lib/subscription_target.h>

class Application : public dcm_lib::ConnectivityManagerLibraryListener {
  // ConnectivityManagerLibraryListener interface
 public:
  void OnNetworkAdded(const dcm_lib::Network& network) {
    std::cout << "Network Added : " << network.get_network_name() << " : "
              << network.get_network_type() << std::endl;
  }
  void OnNetworkRemoved(const dcm_lib::Network& network) {
    std::cout << "Network Removed : " << network.get_network_name() << " : "
              << network.get_network_type() << std::endl;
  }
  void OnCustomNetworkAdded(dcm_lib::SubscriptionTarget target,
                            const std::string& value) {
    std::cout << "Custom network Added. Target: " << target
              << ". Value: " << value << std::endl;
  }
  void OnCustomNetworkRemoved(dcm_lib::SubscriptionTarget target,
                              const std::string& value) {
    std::cout << "Custom network Removed. Target: " << target
              << ". Value: " << value << std::endl;
  }

  void NetworkStateChanged() {}
};

void* SubscribeNetworkListener(void* dcml_ptr) {
  pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, NULL);
  dcm_lib::DistributedConnectivityManagerLibrary* dcml =
      reinterpret_cast<dcm_lib::DistributedConnectivityManagerLibrary*>(dcml_ptr);
  dcml->ProcessNotificationLoop();
  pthread_testcancel();
  return 0;
}

int main() {
  std::string error;
  dcm_lib::DistributedConnectivityManagerLibrary* dcml =
      dcm_lib::DcmLib::Init("./libdcml.so", &error);
  if (!dcml || error != "") {
    std::cout << "Error loading library : " << error << std::endl;
    return 1;
  }

  std::vector<dcm_lib::Network> networks = dcml->GetAvailableNetworks();
  std::cout << "Networks:" << std::endl;
  std::vector<dcm_lib::Network>::iterator it = networks.begin();
  for (; it != networks.end(); it++) {
    const dcm_lib::Network& network = *it;
    std::cout << network.get_network_name() << " : "
              << network.get_network_type() << std::endl;
  }

  dcm_lib::Connection* connection = dcml->GetConnection();
  if (!connection) {
    std::cout << "NULL ptr" << std::endl;
    return 1;
  }

  Application application;
  pthread_t subscribe_network_thread;
  bool is_network_subscribe = false;

  int rcv_input_value;

  do {
    std::cout << "\n\nPress 1 to start download/upload using IVI SDL Virtual "
                 "Network" << std::endl;
    std::cout
        << "Press 2 to start download/upload using IVI SDL GPB"
        << std::endl;
    std::cout
        << "Press 3 to subscribe to network notification"
        << std::endl;
    std::cout << "Press 9 to exit from use cases selection" << std::endl;
    scanf("%d", &rcv_input_value);
    getchar();  // for enter

    switch (rcv_input_value) {
     
      case 1: {
        // IVI SDL to upload/download data
        std::cout << "Press enter key to start download using IVI SDL "
                     "Virtual Network" << std::endl;
        getchar();

        dcm_lib::Connection::RequestParams request_params;
        request_params.use_policy_ = true;
        request_params.low_level_policy_.priority_ = dcm_lib::HP;
        request_params.low_level_policy_.physical_network_ = dcm_lib::IVI_SDL;
        std::string get_response = connection->Get(
            "http://104.41.144.152:3000/view/dl/install_log.txt",
            request_params);
        std::cout << "Response file path : " << get_response << std::endl;

        std::cout << "Press enter key to start IVI SDL "
                     "Virtual Network to upload" << std::endl;
        getchar();

        // upload
        if (!get_response.empty()) {
          bool upload_response =
              connection->Upload("http://104.41.144.152:3000/upload",
                                 get_response, request_params);
          std::string upload_result =
              (1 == upload_response) ? "successuful" : "failed";
          std::cout << "Upload " << upload_result << std::endl;
        }
        break;
      }
      case 2: {
        //IVI SDL GPB to upload/download data
        std::cout << "Press enter key to start download using IVI SDL GPB "
                     "" << std::endl;
        getchar();

        dcm_lib::Connection::RequestParams request_params;
        request_params.use_policy_ = true;
        request_params.low_level_policy_.priority_ = dcm_lib::HP;
        request_params.low_level_policy_.physical_network_ =
            dcm_lib::IVI_SDL_GPB;
        std::string get_response = connection->Get(
            "http://104.41.144.152:3000/view/dl/test1.txt", request_params);
        std::cout << "Response file path : " << get_response << std::endl;

        std::cout << "Press enter key to start IVI SDL GPB "
                     "to upload" << std::endl;
        getchar();

        // upload
        if (!get_response.empty()) {
          bool upload_response =
              connection->Upload("http://104.41.144.152:3000/upload",
                                 get_response, request_params);
          std::string upload_result =
              (1 == upload_response) ? "successuful" : "failed";
          std::cout << "Upload " << upload_result << std::endl;
        }
        break;
      }
      
      case 3: {
        if (!is_network_subscribe) {
          dcml->CustomSubscribe(dcm_lib::WIFI_AP_NAME, "IVDCM");
          dcml->CustomSubscribe(dcm_lib::MAC_ADDRESS, "00:0C:29:CC:2C:9E");
          if(pthread_create(&subscribe_network_thread,
                            NULL, SubscribeNetworkListener, dcml)) {
            std::cerr << "Cannot create thread to subscribe network listener." << std::endl;
          }
          is_network_subscribe = true;
        }
        break;
      }
      case 9: {
        // exit from loop
        break;
      }
    }
  } while (rcv_input_value != 9);

  delete connection;
  pthread_cancel(subscribe_network_thread);
  pthread_join(subscribe_network_thread, NULL);
  dcml->RemoveListener(&application);
  delete dcml;
  return 0;
}
```

####•	Application on Phone:

IVDCM running on phone uses IVDCM processor component to process the messages received from IVI. For example IVDCM processor processes ‘SendIVDCMData’ request received from SDL.  It then uploads/downloads data depending upon the type of request

```c++
public class IVDCMProcessor {

    public static final String LOG_TAG = IVDCMProcessor.class.getSimpleName();

    private static final int BUFFER_SIZE = 1024;

    public static final String USER_AGENT_NAME = "IVDCM Mobile Application";

    private IVDCMProcessor() {

    }

    public static void processSendIVDCMData(@NonNull final VehicleManager vehicleManager,
                                            @NonNull final SendIVDCMData sendIVDCMData) {
        if (isUpload(sendIVDCMData)) {
            Logger.d("TRACE", "UPLOAD");
            processUploadNonIp(vehicleManager, sendIVDCMData);
        } else {
            Logger.d("TRACE", "DOWNLOAD");
            processDownloadNonIp(vehicleManager, sendIVDCMData);
        }
    }

    private static void processUploadNonIp(@NonNull final VehicleManager vehicleManager,
                                           @NonNull final SendIVDCMData sendIVDCMData) {
        Thread workingThread = new Thread(new Runnable() {
            @Override
            public void run() {
                Logger.d(LOG_TAG, " Starting thread");
                final byte[] dataToUpload = sendIVDCMData.getBulkData();
                final String url = sendIVDCMData.getUrl();
                HttpClient httpclient = AndroidHttpClient.newInstance(MainApp.APP_NAME);;
                try {
                    HttpPost httppost = new HttpPost(url);
                    ByteArrayEntity reqEntity = new ByteArrayEntity(dataToUpload);
//                    reqEntity.setContentType("binary/octet-stream");
                    reqEntity.setChunked(true); // Send in multiple parts if needed
                    httppost.setEntity(reqEntity);
                    httppost.addHeader("Expect", "100-continue");
                    Logger.d(LOG_TAG, " Executing put to: " + url);
                    HttpResponse response = httpclient.execute(httppost);
                    String result = EntityUtils.toString(response.getEntity());
                    final String responseInfo = result + "\n" + response.getStatusLine().toString();
                    Logger.d(LOG_TAG, "response from server: " + responseInfo);
                    sendUploadSuccesResponseToSdl(vehicleManager, sendIVDCMData, responseInfo);
                    ((AndroidHttpClient)httpclient).close();
                } catch (Exception e) {
                    e.printStackTrace();
                    sendUploadFailedResponseToSdl(vehicleManager, sendIVDCMData, e.getMessage());
                } finally {
                    if (httpclient != null) {
                        ((AndroidHttpClient)httpclient).close();
                    }
                }
            }
        });
        workingThread.start();
    }

    private static void processDownloadNonIp(@NonNull final VehicleManager vehicleManager,
                                             @NonNull final SendIVDCMData sendIVDCMData) {

        Thread workingThread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    URL url = new URL(sendIVDCMData.getUrl());
                    Logger.d(LOG_TAG, " Starting download url: " + url);
                    HttpURLConnection httpConn = (HttpURLConnection) url.openConnection();
                    int desiredResponseCode = HttpURLConnection.HTTP_OK;
                    if (hasOffset(sendIVDCMData)) {
                        Logger.d(LOG_TAG, "OFFSET: " + sendIVDCMData.getOffset());
                        httpConn.setRequestProperty("Range", "bytes=" + sendIVDCMData.getOffset() + "-");
                        desiredResponseCode = HttpURLConnection.HTTP_PARTIAL;
                    }
                    int responseCode = httpConn.getResponseCode();
                    if (responseCode == desiredResponseCode) {
                        String fileName = "";
                        String disposition = httpConn.getHeaderField("Content-Disposition");
                        String contentType = httpConn.getContentType();
                        int contentLength = httpConn.getContentLength();
                        //TODO just for logging
                        if (disposition != null) {
                            // extracts file name from header field
                            int index = disposition.indexOf("filename=");
                            if (index > 0) {
                                fileName = disposition.substring(index + 10,
                                        disposition.length() - 1);
                            }
                        } else {
                            // extracts file name from URL
                            fileName = sendIVDCMData.getUrl().substring(sendIVDCMData.getUrl().
                                            lastIndexOf("/") + 1,
                                    sendIVDCMData.getUrl().length());
                        }

                        Logger.d(LOG_TAG, "Content-Type = " + contentType);
                        Logger.d(LOG_TAG, "Content-Disposition = " + disposition);
                        Logger.d(LOG_TAG, "Content-Length = " + contentLength);
                        Logger.d(LOG_TAG, "fileName = " + fileName);

                        InputStream inputStream = httpConn.getInputStream();

                        int bytesDownloaded = 0;
                        int bytesRead = -1;
                        byte[] buffer = new byte[BUFFER_SIZE];
                        while ((bytesRead = inputStream.read(buffer)) != -1) {
                            //TODO should create and send IVDCM responses here
                            Logger.d(LOG_TAG, " Downloaded bytes: " + buffer.length);
                            bytesDownloaded += buffer.length;
                            //response to sdl
                            sendDownloadPartialContentResponse(vehicleManager, sendIVDCMData, buffer);
                            Logger.d(LOG_TAG, "Downloaded: " + bytesDownloaded);
                        }
                        //response to sdl
                        sendDownloadSuccessResponseToSdl(vehicleManager, sendIVDCMData);

                        inputStream.close();

                        Logger.d(LOG_TAG, "File downloaded");
                    } else {
                        Logger.d(LOG_TAG, "No file to download. Server replied HTTP code: " + responseCode);
                        final String errorMessage = "No file to download. Server replied HTTP code: " + responseCode;
                        sendDownloadFailureToSdl(vehicleManager, sendIVDCMData, errorMessage);
                    }
                    httpConn.disconnect();
                } catch (MalformedURLException e) {
                    e.printStackTrace();
                    sendDownloadFailureToSdl(vehicleManager, sendIVDCMData, e.getMessage());
                } catch (IOException e) {
                    e.printStackTrace();
                    sendDownloadFailureToSdl(vehicleManager, sendIVDCMData, e.getMessage());
                }
            }
        });
        workingThread.start();
    }
	    private static void sendDownloadPartialContentResponse(@NonNull final VehicleManager vehicleManager,
                                                           @NonNull final SendIVDCMData sendIVDCMData,
                                                           @NonNull final byte[] data) {
        if (vehicleManager == null) {
            Logger.d(LOG_TAG, " Vehicle manager is null. Could not send response to SDL");
            return;
        }
        SendIVDCMDataResponse result = new SendIVDCMDataResponse();
        result.setCorrelationId(sendIVDCMData.getCorrelationId());
        result.setSendDataResult(SendIVDCMDataResponse.PARTIAL_CONTENT);
        result.setBulkData(data);
        Logger.d(LOG_TAG, "sendDataResult: " + result.getSendDataResult());
        vehicleManager.sendRPCMessage(result);
    }

    private static void sendDownloadSuccessResponseToSdl(@NonNull final VehicleManager vehicleManager,
                                                         @NonNull final SendIVDCMData sendIVDCMData) {
        if (vehicleManager == null) {
            Logger.d(LOG_TAG, " Vehicle manager is null. Could not send response to SDL");
            return;
        }
        SendIVDCMDataResponse result = new SendIVDCMDataResponse();
        result.setCorrelationId(sendIVDCMData.getCorrelationId());
        result.setSendDataResult(SendIVDCMDataResponse.SUCCESS);
        Logger.d(LOG_TAG, "sendDataResult: " + result.getSendDataResult());
        vehicleManager.sendRPCMessage(result);
    }
    private static void sendDownloadFailureToSdl(@NonNull final VehicleManager vehicleManager,
                                                 @NonNull final SendIVDCMData sendIVDCMData,
                                                 @NonNull final String errorMessage) {
        if (vehicleManager == null) {
            Logger.d(LOG_TAG, " Vehicle manager is null. Could not send response to SDL");
            return;
        }
        SendIVDCMDataResponse result = new SendIVDCMDataResponse();
        result.setCorrelationId(sendIVDCMData.getCorrelationId());
        result.setSendDataResult(SendIVDCMDataResponse.GENERIC_ERROR);
        result.setInfo(errorMessage);
        Logger.d(LOG_TAG, "sendDataResult: " + result.getSendDataResult());
        vehicleManager.sendRPCMessage(result);
    }

```
####•	Additions to Mobile_API:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<interface name="IVDCM" version="1.1" date="2015-06-22">
   <enum name="FunctionID" internal_scope="base">
      <description>Enumeration linking function names with function IDs in AppLink protocol.</description>
      <description>Assumes enumeration starts at value 0.</description>
      <element name="SendIVDCMDataID" value="11001" hexvalue="2AF9" />
      <element name="OnInternetStateChangedID" value="11002" hexvalue="2AFA" />
	  <element name="OnIVDCMIPData" value="11003" hexvalue="2AFB" />
   </enum>
   <enum name="SendDataResult">
     <description>Enumeration defines the result Codes applicable to data sending requests</description>
     <element name="SUCCESS" value="0" />
	 <element name="TIMED_OUT" value="1" />
	 <element name="GENERIC_ERROR" value="2" />
	 <element name="REQUESTED_CONDITION_NOT_AVAILABLE" value="3" />
	 <element name="PARTIAL_CONTENT" value= "4" />
   </enum>
   <function name="SendIVDCMData" functionID="SendIVDCMDataID" messagetype="request">
      <description>Request from IVDCM to mobile device. 
	  Binary data can be included in hybrid part of message in case of sending IP data to SDL via VirtualNetwork</description>
      <param name="URL" type="String" minvalue="1" maxvalue="180" mandatory="false">
         <description>Defines a url for uploading non-IP data via GPB. Must be mandatory for a non-IP data transferring.</description>
      </param>
	  <param name="isIPData" type="Boolean" mandatory="true">
         <description> Defines the type of data being send in the request</description>
      </param>
	  <param name="offset" type="Integer" minvalue="0" mandatory="false">
         <description> Defines the IP-data offset in bytes to download the file. Applicable for downloading a portion of the file with a specified offset to file begining. type="Integer" means UInt64 type</description>
      </param>
     <param name="fileName" type="String" minvalue="1" maxvalue="180" mandatory="false">
         <description>Defines a fileName for uploading non-IP data via GPB. Must be mandatory for a non-IP upload data transferring.</description>
     </param>
   </function>
   <function name="SendIVDCMData" functionID="SendIVDCMData" messagetype="response">
      <description>Binary data can be included in hybrid part of message as received data from cloud (non-ip data)</description>
      <param name="sendDataResult" type="SendDataResult" mandatory = "true"> 
         <description>See SendDataResult. Defines the result of request execution.</description>
      </param>
      <param name="info" type="String" maxlength="1000" mandatory="false">
         <description>Provides additional human readable info regarding the result.</description>
      </param>
   </function>
   <function name="OnInternetStateChanged" functionID="OnInternetStateChangedID" messagetype="notification">
      <description>Notifies SDL about the internet connection state change on the mobile device ( becomes active/inactive).</description>
      <param name="internetState" type="Boolean" mandatory="true">
         <description>Internet connection becomes active - internetState: 'true', otherwise 'false'</description>
      </param>
   </function>
    <function name="OnIVDCMIPData" functionID="OnIVDCMIPDataID" messagetype="notification">
      <description>Transfers IP data from mobile app to SDL(as a response on SendIVDCMData IP-type request). Binary data MUST be included in hybrid part of message.</description>
   </function>
</interface>

```

####•	Additions to HMI_API.
```GPB
message SDLRPC {
  enum SDLRPCName {
     ON_INTERNET_STATE = 0;
     SEND_IVDCM_DATA   = 1;
  }
  required MessageType rpc_type = 1;
  required SDLRPCName  rpc_name = 2;
  required bytes       params   = 3;
}

message OnInternetStateChangedNotificationParams {
  required string nic_name = 1;
  required bool   state    = 2;
  required string ip       = 3;
}

message SendIvdcmDataRequestParams{
  required string url         = 1;
  optional bytes upload_data  = 2;
  optional uint64 offset      = 3;
  optional string file_name   = 4;
}

message SendIvdcmDataResponseParams {
  required ResultCode result_code   = 1;
  optional bytes      response_data = 2;
  optional string     info          = 3;
}

```

##Impact on existing code
###SDL changes:
*	Addition of IVDCM plugin and IVDCM components.
*	New IVDCM API support.

###Mobile iOS/Android SDK changes:
*	New IVDCM API parameters support.
