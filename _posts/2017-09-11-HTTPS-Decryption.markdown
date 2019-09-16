---
layout: post
title:  "HTTPS Decryption"
date:   2017-09-11 17:45:55
categories: HTTPS Decryption
---
## HTTPS Decryption 之 证书重签##
现如今国外网站普遍部署了HTTPS来加密流量, 国内的网站也在加紧部署中, 主流的购物，金融，搜索等知名网站都默认将流量导向HTTPS.

许多知名厂商的企业级web security产品，都需要分析HTTP traffic。对于HTTPS 的websites, 就涉及到HTTPS Decryption的问题。如[websense](https://www.websense.com/content/support/library/web/hosted/admin_guide/ssl_enable.aspx), [IWSVA](https://docs.trendmicro.com/all/ent/iwsva/v5.5/en-us/iwsva_5.5_olh/about_https_decryption.htm), [WSA](https://www.cisco.com/c/en/us/support/docs/security/web-security-appliance/117792-technote-wsa-00.html)等。  **HTTPS Decryption 的实质是MITM(中间人攻击)，其核心是使client信任伪造的证书**。

[What is Man-in-the-Middle Attack?](https://securebox.comodo.com/ssl-sniffing/man-in-the-middle-attack/)   
![MITM](https://securebox.comodo.com/theme/images/man-in-the-middle-attack.png)

### client验证证书的参数 ###
当client browser 收到server证书后， client端需要验证证书内的参数主要有：   

1. **issuer name**
表示发行者， 是签发该证书的中间（CA）证书的 subject name， 通过issuer name循环验证上级证书，直至找到信任证书为止。
2. **subject name**
表示该证书的主体名称， 需要与Client hello 中的host name进行比较， 如果一致则该参数用过验证。
3. **crl**
表示证书吊销列表，通过该证书中提供的吊销链接，下载证书吊销列表， 查看该证书的**serial number**是否在其中。（serial number 是该证书的签发者分配的编号）。

知道client browser验证证书的规则，那么作为中间人如何实现HTTPS Decryption？
答案是中间人充当http proxy 角色， 对于client 来说它是web server, 对于真正的web server 来说它是client browser.

当client browser 发出SSL握手请求后， 中间人劫持到该信息， 并发出自己的SSL握手请求， 首先拿到server证书， 并对证书进行重签， 如果client browser恰巧信任中间人签发证书，那么就实现了HTTPS Decryption 最核心的证书重签部分。

### 证书重签 ###
证书重签的关键是client信任签发证书， 在企业级web security产品中， 比较容易实现部署。 1. 要求企业用户提供购买的CA证书作为产品的签字证书； 2. 要求client端导入产品自签的CA证书。

如果是比较Hack的程序实现HTTPS MITM攻击， 应当将client信任自签CA作为攻击的最核心的目标。  
下面使用openssl 实现证书重签：

```cpp
    bool signCert(X509* ca_cert, EVP_PKEY* ca_pkey, int key_length, X509* org_cert, X509** cert, EVP_PKEY** pkey)   
    {

    	if(ca_cert == NULL || ca_pkey == NULL) {
    	printf("Invalid input parameter\n");
   		return false;
    }

    RSA* new_cert_rsa;
    *cert = NULL;
    *pkey = NULL;

    //取原server证书的issuer
    char issuer[256];
    X509_NAME* xissuer = X509_get_issuer_name(org_cert);
    X509_NAME_oneline(xissuer, issuer, sizeof(issuer));

    //取原server证书的subject name
    char commonName[512];
    X509_NAME *subj = X509_get_subject_name(org_cert);
    X509_NAME_get_text_by_NID(subj, NID_commonName, commonName, sizeof(commonName));

    ASN1_INTEGER *org_serial = X509_get_serialNumber(org_cert);
    char serialStr[64] = {0};
    for(int i = 0; i < org_serial->length; i++) {
    	snprintf(serialStr + 2 * i, 3, "%02x", org_serial->data[i]);
    }

    //生成重签证书的serial number
    char serialNumber[128];
    genSerialNumber(issuer, commonName, serialStr, serialNumber, sizeof(serialNumber));
    printf("generate new serial number: %s\n", serialNumber);

    ASN1_INTEGER * serial = s2i_ASN1_INTEGER(NULL, serialNumber);
    if(serial == NULL) {
    	printf("get serial failed\n");
    	return false;
    }

    //生成重签证书的RSA key-pair
    new_cert_rsa = genRSA(key_length);
    if(new_cert_rsa == NULL) {
    	ASN1_INTEGER_free(serial);
    	printf("gen RSA failed");
    	return false;
    }

    *pkey = EVP_PKEY_new();
    if(*pkey == NULL) {
    	RSA_free(new_cert_rsa);
    	ASN1_INTEGER_free(serial);
    	printf("new private key failed\n");
    	return false;
    }

    if(EVP_PKEY_set1_RSA(*pkey, new_cert_rsa) < 0) {
    	EVP_PKEY_free(*pkey);
    	RSA_free(new_cert_rsa);
    	ASN1_INTEGER_free(serial);
    	pkey == NULL;
    	printf("set private key from RSA failed\n");
    	return false;
    }

    RSA_free(new_cert_rsa);

    int extcount = 0;
    const char* extstr;
    X509_EXTENSION* extension;
    bool subjectAltName = false;

    //取原server证书的SAN
    if((extcount = X509_get_ext_count(org_cert)) > 0) {
    	for(int i = 0; i < extcount; i++) {
    		extension = X509_get_ext(org_cert, i);
    		extstr = OBJ_nid2sn(OBJ_obj2nid(X509_EXTENSION_get_object(extension)));
    		if(!strcmp(extstr, "subjectAltName")) {
    			subjectAltName = true;
    			break;
    		}
   		}
    }

    *cert = X509_new();
    if(*cert == NULL) {
    	ASN1_INTEGER_free(serial);
    	EVP_PKEY_free(*pkey);
    	*pkey = NULL;
    	printf("new cert failed\n");
    	return false;
    }

    //设置伪造证书的SAN
    if(subjectAltName) {
    	X509_add_ext(*cert, extension, -1);
    }

    //设置伪造证书的public key, subject name, version, issuer name等
    if(X509_set_pubkey(*cert, *pkey) <= 0 ||
    X509_set_subject_name(*cert, X509_get_subject_name(org_cert)) <=0 ||
    X509_set_version(*cert, X509_get_version(ca_cert)) <= 0 ||
    X509_set_issuer_name(*cert, X509_get_subject_name(ca_cert)) <= 0 ||
    X509_set_notBefore(*cert, X509_get_notBefore(ca_cert)) <= 0 ||
    X509_set_notAfter(*cert, X509_get_notAfter(ca_cert)) <= 0 ||
    X509_set_serialNumber(*cert, serial) <= 0) {

    	ASN1_INTEGER_free(serial);
    	EVP_PKEY_free(*pkey);
    	*pkey = NULL;

    	X509_free(*cert);
    	*cert = NULL;
    	printf("set cert values failed\n");
    	return false;
    }

    ASN1_INTEGER_free(serial);

    //使用CA 签伪造证书
    if(X509_sign(*cert, ca_pkey, EVP_sha256()) <= 0) {
    	EVP_PKEY_free(*pkey);
    	*pkey = NULL;
    
    	X509_free(*cert);
    	*cert = NULL;
    	printf("sign cert failed\n");
    	return false;
    }

    return true;
    }

```  

完整代码，请参考[yingshulu-github](https://github.com/YingshuLu).
