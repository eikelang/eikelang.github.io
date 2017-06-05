---
layout: post
title: Fun with an old Cisco 7941G IP Phone, Part 1 - Getting started
date: 2017-06-05 15:01 +0200
---
For quite some time I have enjoyed playing around with the SIP protocol. I've run an asterisk installation on a repurposed Seagate Dockstar (go to [http://projects.doozan.com/debian/] if
you're intersted in more details on running Debian on one of those boxes), I've played around with a number of WiFi SIP phones (all of which sucked, unfortunately) and currently my FritzBox 3490 takes care of all my telephony needs.

Anyways, recently I decided I needed another phone and set my mind on obtaining an older Cisco IP phone, flashing that to a SIP firmware and adding an account for that to the FritzBox.
All of which I managed to do eventually, although the ride turned out to be a bit bumpier than I expected.

Most importantly, if you want to set out on the same journey, **do NOT perform a factory reset on the phone**. It is not the most terrible thing in the world but it makes things unnecessarily complicated. 

# What you will need
* Firmware files for the latest SIP firmware for your Cisco phone. I was able to obtain those directly from Cisco by registering for a free account. Some very old files require an additional license but at least in my case I did not need those.
* Depending on the age of the firmware installed on your phone, you'll need one or two older sets of firmware files. You will know in case your phone refuses to update to the newest firmware files in one go. I just grabbed some versions in between the one that was one the phone (which was the Skinny/SCP firmware, but the numbers appear to be in sync) and the one I wanted to upgrade to, and used each on in succession.
* A TFTP server. I just did an ``apt-get install tfptd``. According to most accounts TFTPD32 is the weapon of choice on Windows, but I can claim not first-hand knowledge of that.
* Only if you did a factory-reset of your phone: A DHCP server that allows you to set option 150. TFTPD32 seems to be DHCP and TFTPD rolled into one, so you should be set. On Linux, you can simply use dnsmasq. If, like me, you run some closed box like a FritzBox, you'll need to disable its DHCP server and use another one until the phone is back in working state, hence my recommendation to not perform a factory reset at all.

# Configure the phone to find your TFTP server.
By default, the phone will expect your DHCP server to provide the name/address of a TFTP server through option 150. If your setup allows you to easily do that, 
feel free to just add that option in your DHCP config. Otherwise, go into your phone's settings (checkmark key on the 7941G) and the go to **Network Configuration**, from there to **IPv4 Configuration** and then go down to **Alternate TFTP Server**. You may notice that there is no apparent way to change this, and in fact you will need to unlock the settings in order to be able 
to make any changes. Hit ``**#`` on the keypad and after a second or two the phone will tell you that the settings are unlocked. Now set **Alternate TFTP Server** to **Yes** and on the next line provide the name or IP of your TFTP server.

# Place required files on the TFTP server
* Unzip the firmware ZIP file(s) and place **all** of the extracted files in the tftp server directory. (They should have names like ``SIP41.9-4-2SR3-1S.loads``, ``apps41.9-4-2ES26.sbn``, ``cnu41.9-4-2ES26.sbn`` and so on.) Except for the ``termXX.default.loads`` files the file names don't clash and you can keep the files side-by-side. Unless you did a factory reset (did I mention that I recommend not do do that?) this will not matter.
* Create an XML file called ``SEP<Mac of your phone without colons>.cnf.xml`` i.e. if the MAC address of your phone is ``00:80:41:ae:fd:7e`` the file must be called ``SEP008041AEFD7E.cnf.xml``.Unfortunately it is next to impossible to find documentation on the format of this file, but you can use the following template. Feel free to scour the internet for other example files that might better illustrate the purpose of the individual elements, but this one has served me well so far. Below the full file I will summarize the elements that you need to change to end up with a working configuration.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<device>
   <deviceProtocol>SIP</deviceProtocol>
   <sshUserId>admin</sshUserId>
   <sshPassword>admin</sshPassword>
   <devicePool>
      <dateTimeSetting>
         <!--  Only two digits for the year -->
         <dateTemplate>D.M.YY</dateTemplate> 
         <timeZone>Central Europe Standard/Daylight Time</timeZone>
         <ntps>
            <ntp>
               <!-- IP of an NTP server to use for setting the phone date/time. -->
               <name>192.168.1.1</name>
               <ntpMode>Unicast</ntpMode>
            </ntp>
         </ntps>
      </dateTimeSetting>
      <callManagerGroup>
         <members>
            <member priority="0">
               <callManager>
                  <ports>
                     <ethernetPhonePort>2000</ethernetPhonePort>
                     <sipPort>5060</sipPort>
                     <securedSipPort>5061</securedSipPort>
                  </ports>
                  <!-- IP address of your SIP server -->
                  <processNodeName>192.168.1.1</processNodeName> 
               </callManager>
            </member>
         </members>
      </callManagerGroup>
   </devicePool>

   <commonProfile>
      <phonePassword></phonePassword>
      <backgroundImageAccess>true</backgroundImageAccess>
      <callLogBlfEnabled>2</callLogBlfEnabled>
   </commonProfile>
   <!-- Here you can set the firmware version the phone should load -->
   <loadInformation>SIP41.9-4-2SR3-1S</loadInformation> 

   <vendorConfig>
      <disableSpeaker>false</disableSpeaker>
      <disableSpeakerAndHeadset>false</disableSpeakerAndHeadset>
      <pcPort>0</pcPort>
      <settingsAccess>1</settingsAccess>
      <garp>0</garp>
      <voiceVlanAccess>0</voiceVlanAccess>
      <videoCapability>0</videoCapability>
      <autoSelectLineEnable>0</autoSelectLineEnable>
      <sshAccess>0</sshAccess>
      <sshPort>22</sshPort>
      <webAccess>0</webAccess>
      <spanToPCPort>1</spanToPCPort>
      <loggingDisplay>1</loggingDisplay>
      <loadServer></loadServer>
      <daysDisplayNotActive></daysDisplayNotActive>
      <displayOnTime>03:00</displayOnTime>
      <displayOnDuration>00:01</displayOnDuration>
      <displayIdleTimeout>00:05</displayIdleTimeout>
      <displayOnWhenIncomingCall>1</displayOnWhenIncomingCall>
   </vendorConfig>

   <deviceSecurityMode>1</deviceSecurityMode>
   
   <!-- unused -->
   <authenticationURL>http://192.168.44.1/ciscoauth.php</authenticationURL> 
   <!-- an URL that points to a place that provides phonebook access, unused for now -->  
   <directoryURL>http://192.168.44.1/directory.php</directoryURL> 
   <!-- an URL that points to an image that the phone should show in idle mode, we'll get to that in another post -->
   <idleURL>http://valhalla.fritz.box/idle.xml</idleURL> 
   <!-- timeout (in seconds) after which the idle URI is called -->
   <idleTimeout>60</idleTimeout> 
   <!-- the URL that is called by the phone when hitting the "?" button -->
   <informationURL></informationURL> 

   <messagesURL></messagesURL>
   <proxyServerURL></proxyServerURL>
   <!-- URL that is called by the phone when opening the "services" menu, we'll get to that in another post -->
   <servicesURL>http://valhalla.fritz.box/services.xml</servicesURL> 
   <dscpForSCCPPhoneConfig>96</dscpForSCCPPhoneConfig>
   <dscpForSCCPPhoneServices>0</dscpForSCCPPhoneServices>
   <dscpForCm2Dvce>96</dscpForCm2Dvce>

   <transportLayerProtocol>2</transportLayerProtocol>

   <capfAuthMode>0</capfAuthMode>
   <capfList>
      <capf>
         <phonePort>3804</phonePort>
      </capf>
   </capfList>

   <certHash></certHash>
   <encrConfig>false</encrConfig>

   <sipProfile>
      <sipProxies>
         <backupProxy></backupProxy>
         <backupProxyPort></backupProxyPort>
         <emergencyProxy></emergencyProxy>
         <emergencyProxyPort></emergencyProxyPort>
         <!-- name/IP of the outbound proxy to use -->
         <outboundProxy>192.168.1.1</outboundProxy> 
         <!-- SIP port of the outbound proxy -->
         <outboundProxyPort>5060</outboundProxyPort> 
         <!-- whether to register with the proxy, normally you want "true" here -->
         <registerWithProxy>true</registerWithProxy> 
      </sipProxies>

      <sipCallFeatures>
         <cnfJoinEnabled>true</cnfJoinEnabled>
         <callForwardURI>x--serviceuri-cfwdall</callForwardURI>
         <callPickupURI>x-cisco-serviceuri-pickup</callPickupURI>
         <callPickupListURI>x-cisco-serviceuri-opickup</callPickupListURI>
         <callPickupGroupURI>x-cisco-serviceuri-gpickup</callPickupGroupURI>
         <meetMeServiceURI>x-cisco-serviceuri-meetme</meetMeServiceURI>
         <abbreviatedDialURI>x-cisco-serviceuri-abbrdial</abbreviatedDialURI>
         <rfc2543Hold>false</rfc2543Hold>
         <callHoldRingback>2</callHoldRingback>
         <localCfwdEnable>true</localCfwdEnable>
         <semiAttendedTransfer>true</semiAttendedTransfer>
         <anonymousCallBlock>2</anonymousCallBlock>
         <callerIdBlocking>2</callerIdBlocking>
         <dndControl>0</dndControl>
         <remoteCcEnable>true</remoteCcEnable>
      </sipCallFeatures>

      <sipStack>
         <sipInviteRetx>6</sipInviteRetx>
         <sipRetx>10</sipRetx>
         <timerInviteExpires>180</timerInviteExpires>
         <timerRegisterExpires>3600</timerRegisterExpires>
         <timerRegisterDelta>5</timerRegisterDelta>
         <timerKeepAliveExpires>120</timerKeepAliveExpires>
         <timerSubscribeExpires>120</timerSubscribeExpires>
         <timerSubscribeDelta>5</timerSubscribeDelta>
         <timerT1>500</timerT1>
         <timerT2>4000</timerT2>
         <maxRedirects>70</maxRedirects>
         <remotePartyID>false</remotePartyID>
         <userInfo>None</userInfo>
      </sipStack>

      <autoAnswerTimer>1</autoAnswerTimer>
      <autoAnswerAltBehavior>false</autoAnswerAltBehavior>
      <autoAnswerOverride>true</autoAnswerOverride>
      <transferOnhookEnabled>false</transferOnhookEnabled>
      <enableVad>false</enableVad>
      <preferredCodec>none</preferredCodec>
      <dtmfAvtPayload>101</dtmfAvtPayload>
      <dtmfDbLevel>3</dtmfDbLevel>
      <dtmfOutofBand>avt</dtmfOutofBand>
      <alwaysUsePrimeLine>false</alwaysUsePrimeLine>
      <alwaysUsePrimeLineVoiceMail>false</alwaysUsePrimeLineVoiceMail>
      <kpml>3</kpml>

      <natEnabled>false</natEnabled>
      <natAddress></natAddress>

      <stutterMsgWaiting>0</stutterMsgWaiting>

      <callStats>false</callStats>

      <silentPeriodBetweenCallWaitingBursts>10</silentPeriodBetweenCallWaitingBursts>
      <disableLocalSpeedDialConfig>false</disableLocalSpeedDialConfig>

      <startMediaPort>16384</startMediaPort>
      <stopMediaPort>32766</stopMediaPort>

     <voipControlPort>5060</voipControlPort>
     <dscpForAudio>184</dscpForAudio>
     <ringSettingBusyStationPolicy>0</ringSettingBusyStationPolicy>
     <dialTemplate>dialplan.xml</dialTemplate>

      <phoneLabel>Office</phoneLabel>	
      <sipLines>
         <!-- Configures the soft function keys to the right of the display
            <featureID>9</featureID> use this for primary lines (i.e. selecting the line for an outbound call)
            <featureID>2</featureID> Speed dial 
            -->
         <line button="1">
            <featureID>9</featureID>
            <!-- Text displayed next to the soft key -->
            <featureLabel>Line 1</featureLabel>   
            <!-- name of the SIP account associated to this line. Attention FritzBox users: must be identical to authName -->
            <name>office</name> 
            <displayName>620</displayName>
            <contact>620</contact>
             <!-- Leave this set to USECALLMANAGER. If you expicitly enter the SIP proxy here again, things will not work. USECALLMANAGER
                  uses the proxy configured in the <callManagerGroup> section -->
            <proxy>USECALLMANAGER</proxy>
            <port>5060</port>
            <autoAnswer>
               <autoAnswerEnabled>2</autoAnswerEnabled>
            </autoAnswer>
            <callWaiting>3</callWaiting>

            <!-- Authentication name of the SIP user. Must be identical to "name" for FRITZ!box -->
            <authName>office</authName> 
            <!-- Authentication password of the SIP user. -->
            <authPassword>SECRET_PASSWORD</authPassword> 

            <sharedLine>false</sharedLine>
            <messageWaitingLampPolicy>1</messageWaitingLampPolicy>
            <!-- the extension to dial when the user presses the "messages" key -->
            <messagesNumber>**600</messagesNumber> 
            <ringSettingIdle>4</ringSettingIdle>
            <ringSettingActive>5</ringSettingActive>

            <forwardCallInfoDisplay>
               <callerName>true</callerName>
               <callerNumber>true</callerNumber>
               <redirectedNumber>false</redirectedNumber>
               <dialedNumber>true</dialedNumber>
            </forwardCallInfoDisplay>
         </line>
         
         <!-- You can freely assign function to buttons, here is an example for speed dial on button "6" -->
         <line button="6"> 
            <featureID>2</featureID>
            <featureLabel>Emergency Call</featureLabel>
            <speedDialNumber>911</speedDialNumber>
         </line> 
       
      </sipLines>
   </sipProfile>
</device>
```

# Key configuration elements
* ``loadInformation``: This element tells your phone which firmware to load. Each firmware ZIP includes a file like ``SIP41.9-4-2SR3-1S.loads``. Use that name, minus the ".loads" extension, to identify the desired firmware version. (As mentioned above, if your phone refuses to go straight to the desired version, try a lower version first and approach the desired version step-by-step.)
* ``processNodeName`` inside the ``callManager`` element: The name/IP of your SIP server.
* ``sipProfile/sipProxies``: Of note in this element are ``outboundProxy``, ``outboundProxyPort`` and ``registerWithProxy`` to configure the IP/name and port of your outbound SIP proxy and to enable proxy usage. 
* ``sipLines``: This section is of course important in order to be able to make any calls. Make sure you configure at least one ``line`` block, be sure to leave in ``<proxy>USECALLMANAGER</proxy>`` and substitute the corect values for ``name``, ``authName`` (watch out: the FritzBox won't work if the two are not identical) and ``authPassword``. Set ``messagesNumber`` so it points to the correct extension for accessing your voice messages.

# Wrapping things up
If everything went according to plan you should now be able to make and receive calls from your Cisco IP phone. If you want to get rid of the log messages concerning the dialplan you can simply put an empty dialplan on the TFTP server, like so:
```xml
<dialplan>
</dialplan>
```

I will follow up with some more posts on:
* Configuring an idle url and having fun with the services menu.
* Adding custom ringtones.
* Adding some plumbing to allow the phone to use a CalDAV server as a directory back-end.

# References
In the process of getting my phone to work the way I want it to, I pulled together information from various sources. If this guide is of any use to you it is by virtue of the following sources. The list is non-exhaustive and in no particular order:
* (German) [Marvin Menzerath on getting a Cisco 7940/7960 working on a FritzBox|https://blog.marvin-menzerath.de/artikel/cisco-ip-phone-7940-7960-mit-fritzbox-verwenden/] 
* (German) [Getting a Cisco 7970 to work with a FritzBox, plus some general configuration info|http://www.arbeitsplatzvernichtung-durch-outsourcing.de/marty44/fritzcisco7970.html]
* (German) [Getting a Cisco 7961G to work with a FritzBox|https://ctx4tom.wordpress.com/2014/03/16/cisco-ip-phone-an-fritzbox-teil-1-sip-firmware-aufspielen/]
* [How to unlock the settings on Cisco IP Phones|https://supportforums.cisco.com/discussion/10936506/unlocking-settings-7945-sip-phone]
* [Installing SIP firmware on a Cisco IP Phone|http://www.pearsonitcertification.com/articles/article.aspx?p=1320201&seqNum=6]
* [Rebooting a Cisco IP Phone|http://www.runpcrun.com/rebootciscophone]
* (Bonus) [Fixing the hook switch on Cisco IP Phones|http://www.nelsonet.net/wordpress/blog/2009/05/28/cisco-7941-and-7961-phone-handset-will-not-pick-up-or-answer/]
* [The VoIP info wiki has some documentation regarding the phone configuration|https://www.voip-info.org/wiki/view/Standalone+Cisco+7941/7961+without+a+local+PBX] 
