``` java
package com.netease.ysf.service;

import com.netease.ysf.common.SelfIpccInfoDefine;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.sip.*;
import javax.sip.address.AddressFactory;
import javax.sip.header.HeaderFactory;
import javax.sip.message.MessageFactory;
import java.net.*;
import java.util.Enumeration;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Properties;


/**
 * 工程启动后自动创建SipFactory、sipStack、addressFactory、headerFactory、messageFactory、sipProviderMap
 * 对外提供addressFactory、headerFactory、messageFactory、sipProvider和Port
 * @author 廉莹
 */

@Component
@Slf4j
public class SipProviderPool {
    private static Properties prop;
    private static javax.sip.SipFactory sipFactory;
    private static SipStack sipStack;
    @Getter
    private static AddressFactory addressFactory;
    @Getter
    private static HeaderFactory headerFactory;
    @Getter
    private static MessageFactory messageFactory;
    private static HashMap<Integer,SipProvider> sipProviderMap = new HashMap<>();
    private static HashSet<Integer> portSet = new HashSet<>();
    @Getter
    static String localHost;
    static int localPort = 5070;
    static int step = 10;

    static{
        //todo: 自动获取本地ip地址的代码
        try {
            Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
            while (interfaces.hasMoreElements()) {
                NetworkInterface networkInterface = interfaces.nextElement();
                if (networkInterface.isUp() && !networkInterface.isLoopback()) {
                    Enumeration<InetAddress> addresses = networkInterface.getInetAddresses();
                    while (addresses.hasMoreElements()) {
                        InetAddress address = addresses.nextElement();
                        if (address instanceof Inet4Address) {
                            localHost = address.getHostAddress();
                        }
                    }
                }
            }
        } catch (SocketException e) {
            throw new RuntimeException(e);
        }
        log.info("localHost:"+localHost);
        //localHost = "10.199.146.223";

        sipFactory = javax.sip.SipFactory.getInstance();
        sipFactory.setPathName("gov.nist");
        prop = new Properties();
        prop.setProperty("javax.sip.STACK_NAME", "mPhone");
        prop.setProperty("javax.sip.IP_ADDRESS", localHost);
        prop.setProperty("gov.nist.javax.sip.TRACE_LEVEL", "32");
        prop.setProperty("gov.nist.javax.sip.DEBUG_LOG", "jainsiplogs//sipclientdebug.txt");
        prop.setProperty("gov.nist.javax.sip.SERVER_LOG", "jainsiplogs//sipclientlog.txt");
        try {
            headerFactory = sipFactory.createHeaderFactory();
            addressFactory = sipFactory.createAddressFactory();
            messageFactory = sipFactory.createMessageFactory();
            createSipProvider(localPort);
        } catch (PeerUnavailableException e) {
            throw new RuntimeException(e);
        }catch (Exception e){
            e.printStackTrace();
            throw e;
        }
    }

    private static void createSipProvider(Integer initialPort){

            int count = 0;

            while(count<step){
                try {
                    SipStack sipStack = sipFactory.createSipStack(prop);
                    ListeningPoint listeningPointUdp = sipStack.createListeningPoint(localPort, "udp");
                    SipProvider sipProvider = sipStack.createSipProvider(listeningPointUdp);
                    if (sipProvider != null) {
                        sipProviderMap.put(localPort, sipProvider);
                        portSet.add(localPort);
                        localPort++;
                        count++;
                    }
                } catch (TransportNotSupportedException | ObjectInUseException e){
                    throw new RuntimeException(e);
                } catch (InvalidArgumentException e){
                    //如果端口号被占用
                    localPort++;
                } catch (PeerUnavailableException e) {
                    throw new RuntimeException(e);
                }
            }
    }

    public static Integer getListeningPort(){
        if(portSet.isEmpty()) {
            try{
                createSipProvider(localPort+1);
            }catch (Exception e){
                throw e;
            }
        }

        Integer port = portSet.iterator().next();
        portSet.remove(port);
        return port;
    }

    public static SipProvider getSipProvider(Integer port){
        return sipProviderMap.get(port);
    }

    public static void releaseSipProvider(Integer port){
            portSet.add(port);
    }

    public static void main(String[] args) {
        int port = getListeningPort();
        SipProvider provider = getSipProvider(port);
    }

}

```