``` java
package com.netease.ysf.service;

import com.netease.ysf.utils.EncodeMethod;
import lombok.Getter;

import javax.sip.*;
import javax.sip.address.Address;
import javax.sip.address.AddressFactory;
import javax.sip.address.SipURI;
import javax.sip.header.*;
import javax.sip.message.MessageFactory;
import javax.sip.message.Request;
import javax.sip.message.Response;
import java.text.ParseException;
import java.util.*;
import java.io.IOException;
import java.io.InputStream;
import java.nio.ShortBuffer;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.util.ArrayList;
import java.util.TooManyListenersException;

import lombok.extern.slf4j.Slf4j;
import org.bytedeco.ffmpeg.avcodec.AVCodecParameters;
import org.bytedeco.ffmpeg.avformat.AVFormatContext;
import org.bytedeco.ffmpeg.avformat.AVStream;
import org.bytedeco.ffmpeg.global.avcodec;
import org.bytedeco.ffmpeg.global.avutil;
import org.bytedeco.javacv.*;
import org.springframework.core.io.ClassPathResource;

/**
 * 提供模拟坐席、模拟访客的接口
 * 已实现功能：
 * 1）sip账号注册及鉴权
 * 2）模拟外呼、接听、挂断
 * 3）挂断只能通过外呼的电话，对于接听的电话不支持挂断
 * 4) 不支持通话中再次接听或者拨打电话
 * 5) 支持指定ip地址，不支持指定port，port是自动分配的
 * @author 廉莹,董烨
 * @version 1.0
 */

@Slf4j
public class BasicPhone implements SipListener{

    private String fullname;
    private String username;
    private String password;
    @Getter
    private String localHost;
    @Getter
    private Integer localPort;
    private String localDomain;
    private String proxyHost;
    @Getter
    private Integer proxyPort;
    private String proxyDomain;
    private boolean isAnswer;
    private javax.sip.SipFactory sipFactory;
    @Getter
    private SipStack sipStack;
    @Getter
    private AddressFactory addressFactory;
    @Getter
    private HeaderFactory headerFactory;
    @Getter
    private MessageFactory messageFactory;
    @Getter
    ListeningPoint listeningPointUdp;
    @Getter
    private SipProvider sipProvider;
    @Getter
    private SipURI requestSipURI;
    @Getter
    private SipURI responseSipURI;
    @Getter
    private FromHeader fromHeader;
    @Getter
    private ToHeader toHeader;
    @Getter
    private ArrayList<ViaHeader> viaHeaderList;
    @Getter
    private CallIdHeader callIdHeader;
    @Getter
    private MaxForwardsHeader maxForwardsHeader;
    @Getter
    private ContactHeader contactHeader;
    @Getter
    private ExpiresHeader expiresHeader;

    //用于主动发bye时识别结束哪个会话，目前支持支持保存一个dialog，因为如果保存dialog，就不能把dialog和对应的phone实例匹配起来
    @Getter
    private final HashMap<String,Dialog> dialogMap = new HashMap<>();

    /**
     *
     * @param fullname  姓名
     * @param username SIPACCOUNT
     * @param password SIPPASSWORD
     * @param env 哪个环境的fs
     * @param isAnswer 哪收到invite是否自动接听
     * @return
     */
    public BasicPhone(String fullname, String username, String password, String env, boolean isAnswer){

        this.isAnswer = isAnswer;
        this.fullname = fullname;
        this.username = username;
        this.password = password;

        String proxyDomain = "";
        switch (env){
            case "test":
                proxyDomain = "59.111.164.247:4160";
                break;
            case "pre":
                proxyDomain = "59.111.222.230:4160";
                break;
            case "online":
                proxyDomain = "59.111.214.250:8700 ";
                break;
        }

        this.proxyDomain = proxyDomain;
        this.proxyHost = proxyDomain.substring(0,proxyDomain.indexOf(":"));
        this.proxyPort = Integer.valueOf(proxyDomain.substring(proxyDomain.indexOf(":")+1));
        headerFactory = SipProviderPool.getHeaderFactory();
        addressFactory = SipProviderPool.getAddressFactory();
        messageFactory = SipProviderPool.getMessageFactory();
        this.localPort = SipProviderPool.getListeningPort();
        sipProvider = SipProviderPool.getSipProvider(localPort);
        this.localHost = SipProviderPool.getLocalHost();
        localDomain = localHost.concat(":").concat(localPort.toString());
        initHeadersInfo();
        sendRegister();
    }
    /**
     *
     * 申请headerFactory、addressFactory、messageFactory、sipProvider等基本SIP组件
     * @param
     * @return
     */
    private void initHeadersInfo(){
        try{
            try {
                requestSipURI = addressFactory.createSipURI(username,proxyDomain);

                //Via: SIP/2.0/UDP 10.221.145.146:5060;branch=z9hG4bK-1588-1-0
                ViaHeader viaHeader = headerFactory.createViaHeader(localHost, localPort, "udp", "branching");
                viaHeaderList = new ArrayList<>();
                viaHeaderList.add(viaHeader);

                //From: 17338114113 <sip:17338114113@10.221.145.146:5060>;tag=1588SIPpTag001
                SipURI fromSipURI = addressFactory.createSipURI(username, localDomain);
                Address fromAddress = addressFactory.createAddress(fromSipURI);
                // tag是随机的
                // fromHeader = headerFactory.createFromHeader(fromAddress, generateRandomString());？
                fromHeader = headerFactory.createFromHeader(fromAddress, "from");

                // 创建一个新的callid
                callIdHeader = sipProvider.getNewCallId();
                // 设置请求最大次数
                maxForwardsHeader = headerFactory.createMaxForwardsHeader(70);

                //Contact: sip:17338114113@10.221.145.146:5060
                SipURI contactURI = addressFactory.createSipURI(username, localHost);
                contactURI.setPort(localPort);
                Address contactAddress = addressFactory.createAddress(contactURI);
                //contactAddress.setDisplayName(username);
                contactHeader = headerFactory.createContactHeader(contactAddress);

                //expires 有效期
                expiresHeader = headerFactory.createExpiresHeader(360000);

            } catch (ParseException | InvalidArgumentException e) {
                e.printStackTrace();
            }

            sipProvider.addSipListener(this);
            sendRegister();
        }catch (TooManyListenersException  e) {
            e.printStackTrace();
        }
    }

    /***
     * 函数作用：释放Phone时，同时释放SipProvider和port资源
     */
    public void destroy(){
        sipProvider.removeSipListener(this);
        SipProviderPool.releaseSipProvider(this.localPort);
    }

    public String getDialog(){
        Set<String> keySet = dialogMap.keySet();
        String dialogID = "";
        if(keySet.iterator().hasNext()){
            dialogID = keySet.iterator().next();
        }
        return dialogID;
    }

    public void deleteDialog(String dialogID){
        Dialog dialog = dialogMap.get(dialogID);
        dialogMap.remove(dialogID);
        dialog.delete();
    }

    /***
     * 函数作用：发送Register请求时调用
     */
    public void sendRegister(){
        try{
            //To: 17338114113 <sip:17338114113@59.111.164.247:4160>
            SipURI toSipURI = addressFactory.createSipURI(username, proxyDomain);
            Address toAddress = addressFactory.createAddress(toSipURI);
            toHeader = headerFactory.createToHeader(toAddress, null);
            CSeqHeader cSeqHeader = headerFactory.createCSeqHeader(1L, Request.REGISTER);
            Request request = messageFactory.createRequest(requestSipURI, Request.REGISTER,
                    callIdHeader, cSeqHeader, fromHeader, toHeader,
                    viaHeaderList, maxForwardsHeader);
            request.addHeader(contactHeader);
            request.addHeader(expiresHeader);
            System.out.println(request);
            sipProvider.sendRequest(request);
        }catch (ParseException | InvalidArgumentException | SipException e){
            e.printStackTrace();
        }
    }

    /***
     * 函数作用：发送Register请求后收到鉴权要求时进行鉴权
     * @param wwwHeader
     */
    private void receiveAuth(WWWAuthenticateHeader wwwHeader){
        String realm = wwwHeader.getRealm();
        String nonce = wwwHeader.getNonce();
        String HA1 = EncodeMethod.getMD5(username +":"+realm+":"+password);
        String HA2 = EncodeMethod.getMD5("REGISTER:sip:"+username+"@"+proxyDomain);
        String resStr = EncodeMethod.getMD5(HA1+":"+nonce+":"+HA2);

        try {

            CSeqHeader cSeqHeader = headerFactory.createCSeqHeader(2L, Request.REGISTER);
            Request request = messageFactory.createRequest(requestSipURI, Request.REGISTER,
                    callIdHeader, cSeqHeader, fromHeader, toHeader,
                    viaHeaderList, maxForwardsHeader);
            request.addHeader(contactHeader);
            request.addHeader(expiresHeader);

            AuthorizationHeader aHeader = headerFactory.createAuthorizationHeader("Digest");

            aHeader.setParameter("realm", realm);
            aHeader.setParameter("nonce", nonce);
            aHeader.setParameter("username", username);
            aHeader.setParameter("uri", addressFactory.createSipURI(username,proxyDomain).toString());
            aHeader.setParameter("response", resStr);
            aHeader.setParameter("algorithm", "MD5");

            request.addHeader(aHeader);

            System.out.println(request);
            sipProvider.sendRequest(request);

        }catch (ParseException | InvalidArgumentException | SipException e){
            e.printStackTrace();
        }
    }

    /***
     * 函数作用：用于caller发送invite请求
     * @param number 被叫号码
     * @return dialogID，用于bye时查询是哪个dialog
     */
    public void sendInvite(String number){
        try {
            requestSipURI = addressFactory.createSipURI(number,proxyDomain);

            //To: 17338114113 <sip:17338114113@59.111.164.247:4160>
            SipURI toSipURI = addressFactory.createSipURI(number, proxyDomain);
            Address toAddress = addressFactory.createAddress(toSipURI);
            toHeader = headerFactory.createToHeader(toAddress, null);

            CSeqHeader cSeqHeader;
            cSeqHeader = headerFactory.createCSeqHeader(1L, Request.INVITE);

            Request inviteRequest;
            inviteRequest = messageFactory.createRequest(requestSipURI, Request.INVITE,
                    callIdHeader, cSeqHeader, fromHeader, toHeader,
                    viaHeaderList, maxForwardsHeader);
            inviteRequest.addHeader(contactHeader);
            inviteRequest.addHeader(expiresHeader);

            // 将 SessionDescription 对象添加到 Request 的消息体中
            ContentTypeHeader contentTypeHeader = headerFactory.createContentTypeHeader("application", "sdp");
            String sd = "v=0\n" +
                    "o=user1 53655765 2353687637 IN IP4 10.221.145.146\n" +
                    "s=-\n" +
                    "c=IN IP4 10.221.145.146\n" +
                    "t=0 0\n" +
                    "m=audio 6000 RTP/AVP 0\n" +
                    "a=rtpmap:0 PCMU/8000";
            inviteRequest.setContent(sd, contentTypeHeader);
            System.out.println(inviteRequest);

            ClientTransaction clientTransaction = sipProvider.getNewClientTransaction(inviteRequest);
            clientTransaction.sendRequest();

        }catch(ParseException | InvalidArgumentException | SipException e){
            e.printStackTrace();
            return;
        }
    }

    /***
     * 函数作用：用于callee收到Invite请求回复100、180、200
     * @param request
     * @param requestEvent
     */
    private void receiveInvite(Request request,RequestEvent requestEvent){

        try{

            log.info("进入 receiveInvite============");

            ServerTransaction serverTransaction = requestEvent.getServerTransaction();
            if(serverTransaction==null){
                log.info("serverTransaction is null");
                serverTransaction = sipProvider.getNewServerTransaction(request);
            }
            Dialog dialog = serverTransaction.getDialog();
            String dialogID = dialog.getDialogId();
            dialogMap.put(dialogID,dialog);

            //回复100
            Response response = messageFactory.createResponse(Response.TRYING, request);
            serverTransaction.sendResponse(response);
            Thread.sleep(200);

            //回复180
            response = messageFactory.createResponse(Response.TRYING, request);
            serverTransaction.sendResponse(response);
            Thread.sleep(200);

            //如果设置接听则回复200接听，否则不处理
            log.info("receiveInvite,isAnswer:"+isAnswer);
            if(isAnswer){

                String sdp = "v=0\n" +
                        "o=- 7538705871719921299 2 IN IP4 " + localHost + "\n" +
                        "s=-\n" +
                        "c=IN IP4 " + localHost + "\n" +
                        "t=0 0\n" +
                        "m=audio 62355 RTP/AVP 8 101\n" +
                        "a=rtpmap:0 PCMU/8000\n" +
                        "a=rtpmap:8 PCMA/8000\n" +
                        "a=rtpmap:101 telephone-event/8000\n" +
                        "a=sendrecv\n" +
                        "a=rtpmap:9 G722/8000\n" +
                        "a=rtcp-mux\n" +
                        "a=setup:active\n" +
                        "a=mid:0\n" +
                        "a=rtcp:9 IN IP4 0.0.0.0\n";

                response = messageFactory.createResponse(Response.OK,request);

                // 创建 Supported 头，添加 Supported 头到响应中
                SupportedHeader supportedHeader = headerFactory.createSupportedHeader("ice,replaces,outbound");
                response.addHeader(supportedHeader);

                // 添加 Contact 头到响应中
                String contactUri = "sip:798709800000161@cc.qiyukf.com;transport=wss";
                Address contactAddress = addressFactory.createAddress(contactUri);
                ContactHeader contactHeader = headerFactory.createContactHeader(contactAddress);
                response.addHeader(contactHeader);

                ContentTypeHeader contentTypeHeader = headerFactory.createContentTypeHeader("application", "sdp");
                ContentLengthHeader contentLengthHeader = headerFactory.createContentLengthHeader(sdp.getBytes().length);
                response.setContentLength(contentLengthHeader);
                response.setContent(sdp.getBytes(), contentTypeHeader);
                System.out.println("response:\n"+response);

                serverTransaction.sendResponse(response);

            }
        } catch (InvalidArgumentException | SipException | ParseException |InterruptedException e) {
            throw new RuntimeException(e);
        }

    }

    /***
     * 函数作用：主动发送bye结束通话
     * @param dialogID
     * @return
     */
    public Integer sendBye(String dialogID){

        //如果通话已经结束，直接返回
        if(!dialogMap.containsKey(dialogID))  return 200;

        try{
            Dialog dialog = dialogMap.get(dialogID);
            Request byeReq = dialog.createRequest(Request.BYE);
            ClientTransaction clientTran = sipProvider.getNewClientTransaction(byeReq);
            dialog.sendRequest(clientTran);
        } catch (SipException e){
            e.printStackTrace();
            return 8000;
        }
        return 200;
    }

    /***
     * 函数作用：收到bye请求时回复200
     * @param request
     * @param requestEvent
     */
    private void receiveBye(Request request,RequestEvent requestEvent){

        try {
            ServerTransaction serverTransaction = requestEvent.getServerTransaction();
            if (serverTransaction == null) {
                serverTransaction = sipProvider.getNewServerTransaction(requestEvent.getRequest());
            }

            Response response = messageFactory.createResponse(Response.OK, request);
            serverTransaction.sendResponse(response);
            System.out.println("send 200 for bye:" + response);

            Dialog dialog = requestEvent.getDialog();
            String dialogID = dialog.getDialogId();
            dialogMap.remove(dialogID);
            dialog.delete();
            
        } catch (SipException | InvalidArgumentException | ParseException e) {
            e.printStackTrace();
        }
    }

    private void sendAck(ResponseEvent responseEvent){
        Transaction clientTransaction = responseEvent.getClientTransaction();
        Dialog dialog = clientTransaction.getDialog();
        try {
            Request ackRequest = dialog.createRequest(Request.ACK);
            //sendAck()会卡住不知道为什么，不能在调用sendAck之后执行其他的
            dialog.sendAck(ackRequest);
            // 接下来是 media transmission

            //startMedia();
        } catch (SipException e) {
            throw new RuntimeException(e);
        }
    }

    /***
     * 函数作用：收到ACK后开始媒体传输
     * @param request
     * @param requestEvent
     */
    private void receiveAck(Request request,RequestEvent requestEvent){
        try {
            // 接下来是 media transmission
            Thread.sleep(100000);
            startMedia();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    /***
     * 函数作用：发送invite并收到200回复后，向uas发送ack完成三次握手
     * @param response
     * @param responseEvent
     */
    private void receiveOK(Response response,ResponseEvent responseEvent){

        String method = ((CSeqHeader)response.getHeader(CSeqHeader.NAME)).getMethod();
        log.info("receiveOK method:"+method);
        Dialog dialog = responseEvent.getDialog();
        String dialogID = dialog.getDialogId();
        switch (method){
            case Request.INVITE:

                log.info("receiveOK method:"+method+"dialogID:"+dialogID);
                dialogMap.put(dialogID,dialog);
                //如果是发送invite后收到的200回复，需要回复ack完成三次握手
                sendAck(responseEvent);
                break;
            case Request.BYE:
                //如果是发送bye后收到的200回复
                deleteDialog(dialogID);
                break;
        }
    }

    @Override
    public void processRequest(RequestEvent requestEvent) {

        Request request = requestEvent.getRequest();
        System.out.println("processRequest执行,request内容是: \n"+request);

        String method = request.getMethod();
        log.info("request method:"+method);
        switch(method){
            case Request.INVITE:
                log.info("进入 receiveInvite************");
                receiveInvite(request,requestEvent);
                break;
            case Request.BYE:
                receiveBye(request,requestEvent);
                break;
            case Request.ACK:
                receiveAck(request,requestEvent);
                break;
        }
    }

    @Override
    public void processResponse(ResponseEvent responseEvent) {

        System.out.println("processResponse执行");

        Response response = responseEvent.getResponse();

        System.out.println("Response is :"+response);

        Integer statusCode = response.getStatusCode();
        System.out.println("返回码:"+statusCode);

        if ( statusCode == Response.TRYING) {
            System.out.println("The response is 100 response.");
            return;
        }

        //收到发送invite后uas回复的200
        if (statusCode == Response.OK) {
            receiveOK(response,responseEvent);
        }

        //收到发送register后uas返回的鉴权信息
        WWWAuthenticateHeader wwwHeader = (WWWAuthenticateHeader) response.getHeader(WWWAuthenticateHeader.NAME);
        if(null != wwwHeader) {
            receiveAuth(wwwHeader);
            return ;
        }

    }

    @Override
    public void processTimeout(TimeoutEvent timeoutEvent) {

    }

    @Override
    public void processIOException(IOExceptionEvent exceptionEvent) {

    }

    @Override
    public void processTransactionTerminated(TransactionTerminatedEvent transactionTerminatedEvent) {

    }

    @Override
    public void processDialogTerminated(DialogTerminatedEvent dialogTerminatedEvent) {

    }

    /**
     * 读取指定的wav文件，推送到媒体服务器
     * @param sourceFilePath 视频文件的绝对路径:暂时没用
     * @param PUSH_ADDRESS 推流地址：暂时没用
     * @throws Exception
     */
    private static void pushWav(String sourceFilePath, String PUSH_ADDRESS,String mediaDomain) throws Exception {
        /* fs回的183携带的sdp
            v=0
            o=YSF-SWITCH 1695073267 1695073268 IN IP4 59.111.166.96
            s=YSF-SWITCH
            c=IN IP4 59.111.166.96   （媒体服务器地址）
            t=0 0  （表示开始的时间和结束的时间，在流媒体的直播的时移中见的比较多）
            m=audio 20300 RTP/AVP 0  //m=<媒体类型> <媒体端口> <传输协议> <编码pt值的集合>
            a=rtpmap:0 PCMU/8000     //a=rtpmap:<pt值> <音频协议> <采样率>
            a=ptime:20               //a=打包时长
         */

        //SRS的推流地址,SRS_PUSH_ADDRESS = "rtmp://59.111.166.96:11935/live/livestream";
        String SRS_PUSH_ADDRESS = "rtmp://"+mediaDomain;

        // ffmepg日志级别
        avutil.av_log_set_level(avutil.AV_LOG_ERROR);
        FFmpegLogCallback.set();

        try{
            //本地MP4文件的完整路径(wav格式的音频)
            ClassPathResource classPathResource = new ClassPathResource("/static/audio/6e050cbe32b250b59c47765ceb12702e.wav");
            InputStream inputStreamImg = classPathResource.getInputStream();

            // 实例化帧抓取器对象，将文件路径传入
            FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(inputStreamImg);

            log.info("开始初始化帧抓取器");
            long startTime = System.currentTimeMillis();
            // 初始化帧抓取器，例如数据结构（时间戳、编码器上下文、帧对象等），
            // 如果入参等于true，还会调用avformat_find_stream_info方法获取流的信息，放入AVFormatContext类型的成员变量oc中
            grabber.start(true);
            log.info("帧抓取器初始化完成，耗时[{}]毫秒", System.currentTimeMillis()-startTime);

            // grabber.start方法中，初始化的解码器信息存在放在grabber的成员变量oc中
            AVFormatContext avFormatContext = grabber.getFormatContext();

            // 文件内有几个媒体流（一般是视频流+音频流）
            int streamNum = avFormatContext.nb_streams();
            if (streamNum<1) {
                // 没有媒体流就不用继续了
                log.error("文件内不存在媒体流");
                return;
            }
            // 取得视频的帧率
            int frameRate = (int)grabber.getVideoFrameRate();
            log.info("视频帧率[{}]，视频时长[{}]秒，媒体流数量[{}]",
                    frameRate,
                    avFormatContext.duration()/1000000,
                    avFormatContext.nb_streams());

            // 遍历每一个流，检查其类型
            for (int i=0; i< streamNum; i++) {
                AVStream avStream = avFormatContext.streams(i);
                AVCodecParameters avCodecParameters = avStream.codecpar();
                log.info("流的索引[{}]，编码器类型[{}]，编码器ID[{}]", i, avCodecParameters.codec_type(), avCodecParameters.codec_id());
            }

            // 视频宽度
            int frameWidth = grabber.getImageWidth();
            // 视频高度
            int frameHeight = grabber.getImageHeight();
            // 音频通道数量
            int audioChannels = grabber.getAudioChannels();

            log.info("视频宽度[{}]，视频高度[{}]，音频通道数[{}]",
                    frameWidth,
                    frameHeight,
                    audioChannels);

            // 实例化FFmpegFrameRecorder，将SRS的推送地址传入
            FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(SRS_PUSH_ADDRESS,
                    frameWidth,
                    frameHeight,
                    audioChannels);

            // 设置编码格式
            recorder.setVideoCodec(avcodec.AV_CODEC_ID_H264);

            // 设置封装格式
            recorder.setFormat("flv");

            // 一秒内的帧数
            recorder.setFrameRate(frameRate);

            // 两个关键帧之间的帧数
            recorder.setGopSize(frameRate);
            // 设置音频通道数，与视频源的通道数相等
            recorder.setAudioChannels(grabber.getAudioChannels());

            log.info("开始初始化帧抓取器");
            startTime = System.currentTimeMillis();
            // 初始化帧录制器，例如数据结构（音频流、视频流指针，编码器），
            // 调用av_guess_format方法，确定视频输出时的封装方式，
            // 媒体上下文对象的内存分配，
            // 编码器的各项参数设置
            recorder.start();
            log.info("帧录制初始化完成，耗时[{}]毫秒", System.currentTimeMillis()-startTime);

            Frame frame;

            log.info("开始推流");
            startTime = System.currentTimeMillis();
            long videoTS = 0;

            int videoFrameNum = 0;
            int audioFrameNum = 0;
            int dataFrameNum = 0;

            // 假设一秒钟15帧，那么两帧间隔就是(1000/15)毫秒
            int interVal = 1000/frameRate;
            // 发送完一帧后sleep的时间，不能完全等于(1000/frameRate)，不然会卡顿，
            // 要更小一些，这里取八分之一
            interVal/=8;

            // 持续从视频源取帧
            while (null!=(frame=grabber.grab())) {
                videoTS = 1000 * (System.currentTimeMillis() - startTime);

                // 时间戳
                recorder.setTimestamp(videoTS);

                // 有图像，就把视频帧加一
                if (null!=frame.image) {
                    videoFrameNum++;
                }

                // 有声音，就把音频帧加一
                if (null!=frame.samples) {
                    audioFrameNum++;
                }

                // 有数据，就把数据帧加一
                if (null!=frame.data) {
                    dataFrameNum++;
                }

                // 取出的每一帧，都推送到SRS
                recorder.record(frame);

                // 停顿一下再推送
                Thread.sleep(interVal);
            }

            log.info("推送完成，视频帧[{}]，音频帧[{}]，数据帧[{}]，耗时[{}]秒",
                    videoFrameNum,
                    audioFrameNum,
                    dataFrameNum,
                    (System.currentTimeMillis()-startTime)/1000);

            // 关闭帧录制器
            recorder.close();
            // 关闭帧抓取器
            grabber.close();

        }catch (IOException e){
            e.printStackTrace();
            return;
        }

    }

    public void pushAudioBack(String mediaDomain){

        //媒体服务器地址,SRS_PUSH_ADDRESS = "rtmp://59.111.166.96:11935/live/livestream",or "rtmp://59.111.166.96:11935";
        //String SRS_PUSH_ADDRESS = "rtmp://"+mediaDomain;
        String SRS_PUSH_ADDRESS = "lianying.wav";
        int audioChannels = 2;
        int sampleRate = 44100;
        // 初始化音频缓冲区(size = 16 * 音频采样率 * 通道数)
        int audioBufferSize = 16 * sampleRate * audioChannels;

        try{
            // 帧录制器:实例化FFmpegFrameRecorder，将媒体服务器的推送地址传入
            FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(SRS_PUSH_ADDRESS, audioChannels);
            // 码率恒定
            recorder.setAudioOption("crf", "0");
            // 最高音质
            recorder.setAudioQuality(0);
            // 192 Kbps
            recorder.setAudioBitrate(audioBufferSize);
            // 采样率
            recorder.setSampleRate(sampleRate);
            // 立体声
            recorder.setAudioChannels(audioChannels);
            // 编码器
            //recorder.setAudioCodec(avcodec.AV_CODEC_ID_AAC);
            recorder.setAudioCodec(avcodec.AV_CODEC_ID_WAVPACK);
            //本地MP4文件的完整路径(wav格式的音频)
            ClassPathResource classPathResource = new ClassPathResource("/static/audio/6e050cbe32b250b59c47765ceb12702e.wav");
            InputStream inputStreamImg = classPathResource.getInputStream();

            byte[] audioBytes = new byte[audioBufferSize];

            //nBytesRead读取了多少byte的音频数据
            int nBytesRead = inputStreamImg.read(audioBytes, 0, inputStreamImg.available());
            // 因为我们设置的是16位音频格式,所以需要将byte[]转成short[]，byte 1字节、short2字节
            int nSamplesRead = nBytesRead / 2;

            short[] samples = new short[nSamplesRead];
            ByteBuffer.wrap(audioBytes).order(ByteOrder.LITTLE_ENDIAN).asShortBuffer().get(samples);

            ShortBuffer sBuff = null;
            sBuff = ShortBuffer.wrap(samples, 0, nSamplesRead);

            //录制音频数据
            recorder.recordSamples(sampleRate, audioChannels, sBuff);
        }catch (IOException e){
            e.printStackTrace();
        }

    }

    public void pushAudio(String mediaDomain) {
        try{
            ClassPathResource classPathResource = new ClassPathResource("/static/audio/6e050cbe32b250b59c47765ceb12702e.wav");
            InputStream inputStream = classPathResource.getInputStream();
            FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(inputStream);
            grabber.start();
            FrameRecorder recorder = new FFmpegFrameRecorder(mediaDomain, grabber.getAudioChannels());
            recorder.setSampleRate(grabber.getSampleRate());
            recorder.start();

            Frame frame;
            while ((frame = grabber.grabFrame()) != null) {
                recorder.record(frame);
            }
            recorder.stop();
            grabber.stop();
        }catch (FrameGrabber.Exception| FrameRecorder.Exception e){

        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public String getMediaDomianBySDP(String sdp){
        String mediaHost = "";
        String mediaPort = "";
        Iterator<String> lines = Arrays.stream(sdp.split("\\r?\\n")).iterator();
        while(lines.hasNext()){
            String line = lines.next();
            System.out.println("line:"+line);
            if(line.startsWith("c=")){
                mediaHost = line.split(" ")[2];
            }else if(line.startsWith("m=")){
                mediaPort = line.split(" ")[1];
            }
        }
        return mediaHost+":"+mediaPort;
    }

    public void startMedia(){
        //推送音频
        //todo：通过sdp解析媒体地址
        //String uasSDP = request.toString();
        //System.out.println("uasSDP:"+uasSDP);
        //String mediaDomain = getMediaDomianBySDP(uasSDP);
        //System.out.println("mediaDomain:"+mediaDomain);
        //pushAudio(mediaDomain);
        //接受音频
    }

    public static String generateRandomString() {
        int length = 10;
        String characters = "abcdefghijklmnopqrstuvwxyz0123456789";
        Random random = new Random();
        StringBuilder sb = new StringBuilder(length);
        for (int i = 0; i < length; i++) {
            int randomIndex = random.nextInt(characters.length());
            char randomChar = characters.charAt(randomIndex);
            sb.append(randomChar);
        }
        return sb.toString();
    }



}
```