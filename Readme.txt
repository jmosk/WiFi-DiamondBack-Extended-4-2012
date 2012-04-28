General Description - specific details on the use follow

I noticed several shortcomings with the code base of WiFi Shield that I have addressed in a modified version of the WiFi Shield code I have created. If people are interested in these modifications I will put a few modules up at github to share. The application I created has to run endlessly for months on end and recover from any failures and deal with millis() clock wrap arounds that occur every 49.7 days. So several types of failures had to be addressed. My application is periodically going out to read the date/time from the Internet and set the time on the Arduino so I can include the current time in Tweets that I send out if certain sensor conditions occur. These are some of the things I found and addressed:

1) When you submit a GET or POST with <object>.submit(), the TCP connection is made but there is no indication to the application if the connection has successfully been made and if the request completed or not. I have added indicator flags to report when the TCP operation has completed and if the request terminated with a connection being made to the remote server or not. The application can now tell if TCP operations were successful or have failed. 

2) Sometimes a GETrequest or POSTrequest will fail before it is completed because the WiFi connection to the access point is lost during the TCP connection. I have added code to allow the application to create a 'guard timer' on TCP I/O. If the I/O takes more than 2 minutes without completing, the application can make a retransmission request.

3) If repeated retransmission requests occur, the application can now re-initialize the GET or POST object and retry the request again. On some failures the state of the TCP connection could be left active and without this re-initialization, all subsquent requests using that GET or POST object will fail. 

4) On repeated TCP errors I call WiServer.init() again to reset the WiFi hardware. But there is code in WiServer.init() that hangs until the WiFi module connects. If it fails to connect to the access point, you hang in that code forever. So I added a guard timer in that modules which aborts the current attempts and re-initializes the hardware once again to try another time. This loops forever but resets the hardware after each guard timer times out. I have seen cases where it resets the WiFi module several times over a couple of minutes then finally connects and the application is functional once again.

5) I added more DEBUG logic that can be conditionally compiled into the WiServer.cpp and other TCP modules to provide more diagnostic information. I also added a debug.cpp module in support of verbose mode of WiServer to provide additional information such as when your WiFi access point disconnects and reconnects. 

With these changes, my application can run for days on end. There are WiFi disconnects and reconnects, failed TCP/IP connections (loss of access point in the middle of a connection) several WiFi hardware resets that occur and so on. But the retry logic in the application and these changes, insure that the application does not lose any data and continues to operate reliably.

-------------

More details:

A module called DebugPrint.cpp has been added. It contains functions to support the printing of debug messages providing some details of the status of WiServer and G2100 as it is connecting to your access point and remote TCP/IP connections. These two files current have #define DEBUG defined so that progress messages are generated with time stamps/ Comment out theses defines to remove the Debug messages.

Messages have been primarily added into flash memory to save RAM space. The Debug routines have functions to print RAM or flash based messages.

Although several new functions could have been introduced as part of the GET or POST class, to provide for the additional functionality quickly, non-C++ procedures were put in place. It is not as clean as one can make it, but it quickly allowed for the improvement of the operation of the WiFi code.

There new variable have been created:

extern boolean TCP_client_connected;
extern boolean TCP_new_connection;
extern boolean TCP_connection_terminated;

People have noticed that there are times when the TCP stack opens and closes a connection but never actually makes a successful connection to the remote server for a GET or POST. These flags canhelp determine if there was a successful connection or not.

Before calling <Get_or_Post_object>.submit(); first set

   TCP_new_connection = FALSE;                  // No TCP connection established, 1 = connected to remote server
   TCP_connection_terminated = FALSE;
   TCP_client_connected = FALSE;

Set TCP_guard_timer = millis() + 60000; // timeout in 1 minute or some other comfortable maximum time to wait the TCP I/O to complete


To determine when the TCP connection requested has completed its processing, check TCP_connection_terminted to become TRUE. The processing has completed at this point.


If the connection to the remote server was successful, at this point TCP_new_connection will be true. If it is still FALSE, the connection to the remote server never took place. 

During the entire period that the TCP/IP stack is busy servicing a request (from the time the <object>.submit() is called until the TCP/IP connection is closed, TCP_client_connected will be set to TRUE. So your mainline can determine when the stack is busy servicing the request.

The checking of the TCP_guard_timer should be done in your own application. Without considering timer wrap around code, your mainline which keeps calling WiServer.server_task() to process the server code and TCP/IP stack, should also contain code similar to:

	if (TCP_guard_timer < millis() ) {
		< a timeout has occurred >

If a timeout has occurred, the TCP/IP stack has not only failed to connect to the remote, but it has not completed the I/O in progress. There could have been a dropping of the connection to your access point during an attempt to connect, for example, and the stack did not recover from the failure. I did not have the time to venture completely into the workings of the stack to repair the error cases which caused failures. Instead using the timeout logic and minor changes to the stack, I provided a capability to recover from such an error by restarting the TCP/IP stack. So, in the event of a timeout of a TCP/IP operation, you can restart the TCP stack and server and start over without reseting your application by doing the following steps.

First, your Post Request needs to be broken down into two #defines. 'Parameter1' is a #define containing the first 3 parameters of a POSTrequest and Parameter2 is the 4th parameter. For a Get Request, all the parameters are contained in a #define. For example:

#define pPARM1 Send_Msg_IP, 80, "api.msg_server.com", "/cgi-bin/send_a_msg.asp
#define pPARM2 Message_compose_function

POSTrequest SendMsg (pPARM1, pPARM2);

For a Get request it might look like this:

#define gPARM1 Get_time_IP, 80, "api.gettime.com", "/cgi-bin/get_time_of_day.pl
GETrequest GetTime (PARM1);

If there is a timeout, the server and stack may be restarted (and a new connection to your access point established) via calls similar to the following, using the GET and POST above:


	WiServer.init (NULL);
	SendMsg.init (pPARM1);
	SendMsg.setBodyFunc (pPARM2);
	SendMsg.setReturnFunc (same_function_as_original_SendMsg_setting_of_ReturnFunc);
	GetTime.setReturnFunc (same_function_as_original_GetTime_setting_of_ReturnFunc);

The Get or Post object is re-initialized and all flags restarted so the stack restarts cleanly. The init() function restarts the connection to the access point. There was an infinite loop in the code that hung forever if the access point did not connect. That code has been changed to keep attempting to reset the WiFi module and connect to the access point if it fails to reconnect in 45 seconds. 

The version of the code released, has a #define DEBUG in WiServer.cpp and g2100.c. The code will provide progress reports to the USB port when connecting and sending or receiving data and whenever the access point connects or disconnects. To remove these messages, just comment these #defines in these two modules and recompile and link your code.

By using the TCP_new_connection and TCP_connection_terminated flags, your code can now determine of a remote connection was successful and resend the request if there was a failure. If several failures to successfully connect occur in a row, you can execute the reset logic shown above to reset the WiFi chip and stack and try again. I have seen many cases where the 2nd or 3rd attempt succeeds and cases where it was necessary to reset the WiFi chip before normal communications would continue.

If the timeout logic proves that the TCP/IP did not complete before the timeout occured, then flags could be left in the wrong state in the TCP stack and cause subsequent requests to fail. So the best thing is just to re-init the server as shown above, to insure that flags are returned to their proper state.

The DebugPrint.cpp module contains several calls to print debug message with and without time stamps and using messages stored in flash or messages stored in RAM. 

	












