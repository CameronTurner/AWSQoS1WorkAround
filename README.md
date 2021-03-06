# AWS QoS1 Work Around for limited in-flight messages
AWS IoT MQTT Broker has a 100 in-flight message limit which impacts QoS 1 services. This work around moves QoS from TCP to Application level.

**Problem:** AWS MQTT brokers are limited to 100 in-flight messages in a given second. After checking with support is seems that messages above this are dropped.

**What is a in-flight message**
It is a publish event either from the broker or client that is yet to be acknowledged by the other device. 
Typically you see in-flight messages increase in volume when the client device has a poor connection and stops responding to the broker or is slow to respond to the broker (e.g. A small IoT device is busy processing other actions or is blocked working on other code).

**Why is this a problem:**
1. You can publish by default at rates of up to 6000 publishes per second, if a large number of devices are on slow connections or have dropped out before their last check in - the messages will be held 'in-flight' until either:
A. The device times out at the broker end and is force disconnected
B. The messages are sent successfully

**What else does this problem occur with?**
- Be careful with Device shadows - as of time of writing the limit was 10 device shadow publishes in flight per second.
- If a few hundred devices all connect at the same time, the device shadows may not be updated and dropped.

## How do we work around the in-flight message limitation on QoS 1 - while still having the ability to get confirmation that the event was received by the broker and processed?

Step 1: Set your Publish events to AWS to use QoS 0 

  This will vary based on the library you use, the example below is when using the MQTT TLS library from https://github.com/hirotakaster/MQTT-TLS
  
  ```C++
  client.publish("outTopic/message", "hello world");
  ```
  

Step 2: The publish event will need to contain the device ID and a unique message code so the message can be differentiated from other device and other messages that device may send.
e.g. pass the JSON example below.
{
"deviceID":"1243",
"uniqueMsgID":"time"
}

Step 3: Set an action in AWS IoT to 'republish' all events on a given topic when an event is received to a differnet topic
- This bit is easy, go to Actions, subscribe to the topic you want to respond to, then set the action to 'republish events'.
You will need to add the below code into the 'action' topic: 

  ```outReturn/${deviceID}
 ```

Step 4: Go to IAM and select the *ROLE* that you selected to run the re-publish action in AWS IoT.
Set the role to be able to 'access all resources'. There might be a more secure option to allow it access to the 2nd 'republish' so best to look into that depending on your account complexity.

Step 5: Have your device subscribe to a topic where the 'return' message will come back
```client.subscribe(String("outReturn/" + deviceID); 
```
In the above code, we are subscribing to the topic out/myDevicesID - so that way we are not getting the messages for every other device heading our way. If you have 1000+ devices, you don't want all those messages heading off to every device.

Step 6: Have your device compare the unique message ID to the one that was just sent to validate they match.
  ```C++
  if (newUniqueMessageCode == UniqueMessageCode)
  {
  publishEvent_outTopic = 1; //1 is for success
  }
```

Step 7: If the unqiue message ID is not returned within X number of seconds, set your code to republish the message.
This part you'll need to write yourself and embed within your publish as you'll want to handle the 'retry' or 'error handling' of each failure individually. 

e.g. If a SMS event fails, try again for 5 times until the latest uniqueMessageCode is returned. After 5 tries - stop
e.g. If a Valve doesn't close, then rather than try again, instead send a SMS alert advising of the failure and turn off the 2nd valve that might be locally controlled.

## Bonus tip: Discard delayed messages from being republished or actioned
This part has not yet been fully tested - use with caution.<<<

Given we are using QoS 0, it is possible a message we sent earlier may arrive after we have retried a message.
If your message is set to perform an action once and once only (e.g. Send a SMS) you will probably want to discard any late arriving messages. This could be done with time stamps.

In the AWS IoT SQL Select statement set the condition to:

  inputInUnixTimeInSeconds < ((timestamp()/1000.0) + 5)
  A < ((B/C)+D)
  
  In english:
  A = The Variable we pass in the JSON file is called 'inputInUnixTimeInSeconds' , we could call this 'messageSentTime' or 'dog' anything is good. But it must pass the Unix time (see here https://www.epochconverter.com/) If you are using a Particle.io device you get unix time in seconds automatically when you use the code line:
  
    Time.now();
    
   B = The timestamp from the SQL server in miliseconds. Don't worry about time zones as 'unix' time is consistent around the world - it is anchored to "the number of seconds that have elapsed since January 1, 1970 (midnight UTC/GMT)".
   
   C = We need to conver our miliseconds to seconds to do the comparison. You could pass miliseconds in 'A' but it is a very long number and MQTT payload sizes are quite limited, so don't waste space on Miliseconds when seconds could do.
   
   D = Represents the number of seconds (5 seconds) that can elapse since the message was sent - don't set it too low, while both devices are using the same 'time' they may be out by a second or two. If you are using a Particle.io device - remember to run the below code at least every time your start the device and once per day. 
   
  ```C++ 
  Particle.syncTime(); 
  ```
