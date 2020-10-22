# JAVA之微信支付功能实现涉及微信双向证书（PKCS12证书设置与SSL请求封装）现金红包开发流程



### 0.前期准备:

> mchId   微信支付分配的商户号

> apiKey  key为商户平台设置的密钥key

> Appid  微信分配的公众账号ID（企业号corpid即为此appId）。在微信开放平台（open.weixin.qq.com）申请的移动应用appid无法使用该接口。

> API证书  商家在申请微信支付成功后，收到的相应邮件后，可以按照指引下载API证书，也可以按照以下路径下载：微信商户平台(pay.weixin.qq.com)-->账户中心-->账户设置-->API安全 。

### 1.参数加签

1. ◆ 参数名ASCII码从小到大排序（字典序）；

2. ◆ 如果参数的值为空不参与签名；

3. ◆ 参数名区分大小写；

4. ◆ 验证调用返回或微信主动通知签名时，传送的sign参数不参与签名，将生成的签名与该sign值作校验。

5. ◆ 微信接口可能增加字段，验证签名时必须支持增加的扩展字段

   

   #### 参数封装实例:

   ```java
   				 TreeMap<String, String> data = new TreeMap<>();
           data.put("nonce_str", RandomUtil.randomString(31));//随机字符串，不长于32位
           //接口根据商户订单号支持重入， 如出现超时可再调用。
           data.put("mch_billno", mchBillno);
           data.put("mch_id", mchId);
           data.put("wxappid", appId);
           data.put("send_name", "一咻停车");
           data.put("re_openid", userByUserName.getOpenid());
           data.put("total_amount", totalAmount.toString());
           data.put("total_num", "1");
           data.put("wishing", wishing);
           data.put("client_ip", "127.0.0.1");
           data.put("act_name", actName);
           data.put("remark", remark);
           data.put("scene_id", "PRODUCT_5");
           data.put("sign", SignUtil.toSign(data, key));
   ```

   

   #### 对应的加签方法:

   ```java
   public static String toSign(SortedMap<String, String> signParams,String key)   {
           StringBuffer sb = new StringBuffer();
           Set es = signParams.entrySet();
           Iterator it = es.iterator();
           while (it.hasNext()) {
               Map.Entry entry = (Map.Entry) it.next();
               String k = (String) entry.getKey();
               String v = (String) entry.getValue();
               sb.append(k + "=" + v + "&");
           }
           sb.append(key);
           String s = SecureUtil.md5(sb.toString()).toUpperCase();
           return s;
       }
   ```

   

### 2.请求阶段

