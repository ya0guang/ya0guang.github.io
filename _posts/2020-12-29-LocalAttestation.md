---
layout: single
title: SGX Local Attestation 源码分析
date: 2020-12-29 21:10:07
categories: "Notes"
tags:
- Research
- SGX
- Code Analysis
- Crypto
header:
  teaser: 
---

记一个对于Intel SGX Local Attestation Sample Code的一次Comprehensive的analysis。**Warning**：这篇文章十分臭长，超级干燥。

# Overview

[Linux SGX](https://github.com/intel/linux-sgx)样例里面给了Local Attestation的程序[Sample](https://github.com/intel/linux-sgx/tree/master/SampleCode/LocalAttestation)。这里给出了两个Local Attestation的design：一个App下的俩Enclave和俩App对应俩Enclave（这里只分析第二种设计）。从其[Developer Reference](https://download.01.org/intel-sgx/sgx-linux/2.12/docs/Intel_SGX_Developer_Reference_Linux_2.12_Open_Source.pdf), *p95~*, 中可以看到一个关于LA的big map。

![LA Process](/assets/images/SGX/SGX_LA.png)

官方对于这些步骤的explanations可以在[Developer Reference](https://download.01.org/intel-sgx/sgx-linux/2.12/docs/Intel_SGX_Developer_Reference_Linux_2.12_Open_Source.pdf), *p98*，看到，此处不再赘述。接下来分析LA的Implementation。

另一份值得参考的文档是[SGX 101 LA](https://sgx101.gitbook.io/sgx101/sgx-bootstrap/attestation/inter-process-local-attestation)。但是由于其给出的Sample Code大概是好几个版本之前的SDK的，此处仅作参考。

[SGX Explained](https://eprint.iacr.org/2016/086.pdf)

# `sgx_dh_init_session`

该函数在Enclave的`create_session`函数里面被这调用：`status = sgx_dh_init_session(SGX_DH_SESSION_INITIATOR, &sgx_dh_session);`。其中
`SGX_DH_SESSION_INITIATOR`为`sgx_dh_session_role_t`的一个enum，另一个则是`SGX_DH_SESSION_RESPONDER`，代表LA中的两类roles。而`sgx_dh_session`是结构体`sgx_dh_session_t`，其实就是一个简单的`unit8_t`数组，在之后会更详细地介绍。

`Initiator`会调用这个函数，该函数在`sgx_dh.h`这个header里，实现在SDK里面的`ec_dh_cpp`中。

## Implementation
```cpp
sgx_status_t sgx_dh_init_session(sgx_dh_session_role_t role, sgx_dh_session_t* sgx_dh_session)
{
    sgx_internal_dh_session_t* session = (sgx_internal_dh_session_t*)sgx_dh_session;

    if(!session || 0 == sgx_is_within_enclave(session, sizeof(sgx_internal_dh_session_t)))
    {
        return SGX_ERROR_INVALID_PARAMETER;
    }

    if(SGX_DH_SESSION_INITIATOR != role && SGX_DH_SESSION_RESPONDER != role)
    {
        return SGX_ERROR_INVALID_PARAMETER;
    }

    memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));

    if(SGX_DH_SESSION_INITIATOR == role)
    {
        session->initiator.state = SGX_DH_SESSION_INITIATOR_WAIT_M1;
    }
    else
    {
        session->responder.state = SGX_DH_SESSION_STATE_RESET;
    }

    session->role = role;

    return SGX_SUCCESS;
}
```

上面这段代码主要是对`session`初始化：清零了`session`的内存并根据`role`进行了`state`的初始化

### `sgx_internal_dh_session_t` in `sgx_dh_internal.h`
可以看到传入的`sgx_dh_session`被强制转换成了`sgx_internal_dh_session_t`。根据变量名推测是SDK内部对于dh session的representation。其被定义为：

```cpp
typedef struct _sgx_internal_dh_session_t{
    sgx_dh_session_role_t role;             /* Initiator or Responder */
    union{
        // this means an internal session is either an initiator or responder
        sgx_dh_responder_t responder;
        sgx_dh_initator_t  initiator;
    };
} sgx_internal_dh_session_t;

typedef struct _sgx_dh_responder_t{
    sgx_dh_session_state_t   state;	   /*Responder State Machine State */
    sgx_ec256_private_t prv_key;             /* 256bit EC private key */
    sgx_ec256_public_t  pub_key;             /* 512 bit EC public key */
} sgx_dh_responder_t;
 
typedef struct _sgx_dh_initator_t{
    sgx_dh_session_state_t state;    /* Initiator State Machine State */
    union{
        sgx_ec256_private_t prv_key;    /* 256bit EC private key */
        sgx_key_128bit_t smk_aek;    /* 128bit SMK or AEK. Depending on the State */
    };
    sgx_ec256_public_t pub_key;    /* 512 bit EC public key */
    sgx_ec256_public_t peer_pub_key;    /* 512 bit EC public key from the Responder */
    sgx_ec256_dh_shared_t shared_key;
} sgx_dh_initator_t;
```
DH的概述可以参考[wiki](https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B)，此处不再赘述。

# `session_request_ocall`

对其的调用为`status = session_request_ocall(&retstatus, &dh_msg1, &session_id);`。这里图上有一点misleading的地方在于，其实这个ocall并没有直接call到`AppResponder`(Untrusted Code2)，而是先call到了App内的`session_request_ocall`。

## `session_request_ocall`

```cpp
/* Function Description: This is OCALL interface for initiator enclave to get ECDH message 1 and session id from responder enclave
 * Parameter Description:
 *      [input, output] dh_msg1: pointer to ecdh msg1 buffer, this buffer is allocated in initiator enclave and filled by responder enclave
 *      [output] session_id: pointer to session id which is allocated by responder enclave
 * */
extern "C" ATTESTATION_STATUS session_request_ocall(sgx_dh_msg1_t* dh_msg1, uint32_t* session_id)
{
	FIFO_MSG msg1_request;
	FIFO_MSG *msg1_response;
	SESSION_MSG1_RESP * msg1_respbody = NULL;
	size_t  msg1_resp_size;

	msg1_request.header.type = FIFO_DH_REQ_MSG1;
	msg1_request.header.size = 0;

	if ((client_send_receive(&msg1_request, sizeof(FIFO_MSG), &msg1_response, &msg1_resp_size) != 0)
		|| (msg1_response == NULL))
	{
		printf("fail to send and receive message.\n");
		return INVALID_SESSION;
	}

	msg1_respbody = (SESSION_MSG1_RESP *)msg1_response->msgbuf;
	memcpy(dh_msg1, &msg1_respbody->dh_msg1, sizeof(sgx_dh_msg1_t));
	*session_id = msg1_respbody->sessionid;
        free(msg1_response);

	return (ATTESTATION_STATUS)0;
}
```
在这个函数内初始化了一些变量，然后再调用`client_send_receive`以client发送一个`msg1_request`，并且receive `msg1_response`。这个函数是通过socket来完成收发的。

## Server Side

在server端(`AppResponder`)里，一开始的snippet长这样：
```cpp
        switch (message->header.type)
        {
            case FIFO_DH_REQ_MSG1:
            {
                // process ECDH session connection request
                int clientfd = message->header.sockfd;
            
                if (generate_and_send_session_msg1_resp(clientfd) != 0)
                {
                    printf("failed to generate and send session msg1 resp.\n");
                    break;
                }
                // ....
```
得知header是`FIFO_DH_REQ_MSG1`后就开始走DH的`generate_and_send_session_msg1_resp`流程了。完整的代码就不贴了，其主要就是调用EnclaveResponder内的`session_request`来生成response，配上相应的header，然后把消息回传。

## `session_request`

```cpp
//Handle the request from Source Enclave for a session
extern "C" ATTESTATION_STATUS session_request(sgx_dh_msg1_t *dh_msg1,
                          uint32_t *session_id )
{
    dh_session_t session_info;
    sgx_dh_session_t sgx_dh_session;
    sgx_status_t status = SGX_SUCCESS;

    if(!session_id || !dh_msg1)
    {
        return INVALID_PARAMETER_ERROR;
    }
    //Intialize the session as a session responder
    status = sgx_dh_init_session(SGX_DH_SESSION_RESPONDER, &sgx_dh_session);
    if(SGX_SUCCESS != status)
    {
        return status;
    }

    //get a new SessionID
    if ((status = (sgx_status_t)generate_session_id(session_id)) != SUCCESS)
        return status; //no more sessions available

    //Allocate memory for the session id tracker
    g_session_id_tracker[*session_id] = (session_id_tracker_t *)malloc(sizeof(session_id_tracker_t));
    if(!g_session_id_tracker[*session_id])
    {
        return MALLOC_ERROR;
    }

    memset(g_session_id_tracker[*session_id], 0, sizeof(session_id_tracker_t));
    g_session_id_tracker[*session_id]->session_id = *session_id;
    session_info.status = IN_PROGRESS;

    //Generate Message1 that will be returned to Source Enclave
    status = sgx_dh_responder_gen_msg1((sgx_dh_msg1_t*)dh_msg1, &sgx_dh_session);
    if(SGX_SUCCESS != status)
    {
        SAFE_FREE(g_session_id_tracker[*session_id]);
        return status;
    }
    memcpy(&session_info.in_progress.dh_session, &sgx_dh_session, sizeof(sgx_dh_session_t));
    //Store the session information under the corresponding source enclave id key
    g_dest_session_info_map.insert(std::pair<uint32_t, dh_session_t>(*session_id, session_info));

    return status;
}
```
在Responder side也会initialize responder的session。引人注意的是变量`g_session_id_tracker`，其定义为`session_id_tracker_t *g_session_id_tracker[MAX_SESSION_COUNT];`。怀疑是想要满足多个LA sessions的需求。

# `sgx_dh_responder_gen_msg1`

在session建立之后其调用`sgx_dh_responder_gen_msg1`来生成真正的`msg1`。而在这个函数内又调用了真正用来生成`msg1`的`dh_generate_message1`。

## `dh_generate_message1`

```cpp
static sgx_status_t dh_generate_message1(sgx_dh_msg1_t *msg1, sgx_internal_dh_session_t *context)
{
    sgx_report_t temp_report;
    sgx_status_t se_ret;
    sgx_ecc_state_handle_t ecc_state = NULL;

    if(!msg1 || !context)
    {
        return SGX_ERROR_INVALID_PARAMETER;
    }

    //Create Report to get target info which targeted towards the initiator of the session
    se_ret = sgx_create_report(nullptr, nullptr, &temp_report);
    if(se_ret != SGX_SUCCESS)
    {
        return se_ret;
    }

    if (SGX_SUCCESS != (se_ret =
        LAv2_proto_spec.make_target_info(temp_report, msg1->target)))
    {
        return se_ret;
    }

    //Initialize ECC context to prepare for creating key pair
    se_ret = sgx_ecc256_open_context(&ecc_state);
    if(se_ret != SGX_SUCCESS)
    {
        return se_ret;
    }
    //Generate the public key private key pair for Session Responder
    se_ret = sgx_ecc256_create_key_pair((sgx_ec256_private_t*)&context->responder.prv_key,
                                       (sgx_ec256_public_t*)&context->responder.pub_key,
                                       ecc_state);
    if(se_ret != SGX_SUCCESS)
    {
         sgx_ecc256_close_context(ecc_state);
         return se_ret;
    }

    //Copying public key to g^a
    memcpy(&msg1->g_a,
           &context->responder.pub_key,
           sizeof(sgx_ec256_public_t));

    se_ret = sgx_ecc256_close_context(ecc_state);
    if(SGX_SUCCESS != se_ret)
    {
        return se_ret;
    }

    return SGX_SUCCESS;
}
```
在这里，属于caller enclaved的report被`sgx_create_report`创建并存储在`temp_report`。这里奇怪的一点是`make_target_info`发生在`sgx_create_report`之后。而按照`EREPORT`指令的[specification](https://www.felixcloutier.com/x86/ereport)，~~`target_info`应该作为input输入其中。此处有蹊跷~~。

看了后面我才明白，这里`sgx_create_report`生成了关于自己(caller enclave)的一个report，然后通过`make_target_info`导出了report中自己的target info。根据两个结构体的字段不难发现前者包含后者所需要的所有信息。

然后，dh中的`g_a`（被生成出来的keypair中的public ket的部分）被丢在了`msg1`内。相关的其他信息也被存储在了`context`内。最终的`msg1`为`g_a||target_info`。

最后，在AppResponder内，`session_request`返回时带上了需要发送给Initiator的`msg1`。这些信息被发回Initiator

# `sgx_dh_initiator_proc_msg1`

对其的调用在`status = sgx_dh_initiator_proc_msg1(&dh_msg1, &dh_msg2, &sgx_dh_session);`。而`sgx_dh_initiator_proc_msg1`其实是`sgx_LAv2_initiator_proc_msg1`或者`sgx_LAv1_initiator_proc_msg1`的alias。取决于有没有define `LAv2`。

```cpp
template <decltype(dh_generate_message2) gen_msg2>
static sgx_status_t dh_initiator_proc_msg1(const sgx_dh_msg1_t* msg1, sgx_dh_msg2_t* msg2, sgx_dh_session_t* sgx_dh_session)
{
    sgx_status_t se_ret;

    sgx_ec256_public_t pub_key;
    sgx_ec256_private_t priv_key;
    sgx_ec256_dh_shared_t shared_key;
    sgx_key_128bit_t dh_smk;

    sgx_internal_dh_session_t* session = (sgx_internal_dh_session_t*) sgx_dh_session;

    /* ROUTINE CHECKS BEGIN */

    // validate session
    if(!session ||
        0 == sgx_is_within_enclave(session, sizeof(sgx_internal_dh_session_t))) // session must be in enclave
    {
        return SGX_ERROR_INVALID_PARAMETER;
    }

    if( !msg1 ||
        !msg2 ||
        0 == sgx_is_within_enclave(msg1, sizeof(sgx_dh_msg1_t)) ||
        0 == sgx_is_within_enclave(msg2, sizeof(sgx_dh_msg2_t)) ||
        SGX_DH_SESSION_INITIATOR != session->role)
    {
        // clear secret when encounter error
        memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
        session->initiator.state = SGX_DH_SESSION_STATE_ERROR;
        return SGX_ERROR_INVALID_PARAMETER;
    }

    if(SGX_DH_SESSION_INITIATOR_WAIT_M1 != session->initiator.state)
    {
        // clear secret
        memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
        session->initiator.state = SGX_DH_SESSION_STATE_ERROR;
        return SGX_ERROR_INVALID_STATE;
    }

    /* ROUTINE CHECKS END */

    //create ECC context
    sgx_ecc_state_handle_t ecc_state = NULL;
    se_ret = sgx_ecc256_open_context(&ecc_state);
    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }
    // generate private key and public key
    se_ret = sgx_ecc256_create_key_pair((sgx_ec256_private_t*)&priv_key,
                                       (sgx_ec256_public_t*)&pub_key,
                                        ecc_state);
    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }

    //generate shared_key
    se_ret = sgx_ecc256_compute_shared_dhkey(
                                            (sgx_ec256_private_t *)const_cast<sgx_ec256_private_t*>(&priv_key),
                                            (sgx_ec256_public_t *)const_cast<sgx_ec256_public_t*>(&msg1->g_a),
                                            (sgx_ec256_dh_shared_t *)&shared_key,
                                             ecc_state);

    // clear private key for defense in depth
    memset_s(&priv_key, sizeof(sgx_ec256_private_t), 0, sizeof(sgx_ec256_private_t));

    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }

    se_ret = derive_key(&shared_key, "SMK", (uint32_t)(sizeof("SMK") -1), &dh_smk);
    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }

    se_ret = gen_msg2(msg1, &pub_key, &dh_smk, msg2);
    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }

    memcpy(&session->initiator.pub_key, &pub_key, sizeof(sgx_ec256_public_t));
    memcpy(&session->initiator.peer_pub_key, &msg1->g_a, sizeof(sgx_ec256_public_t));
    memcpy(&session->initiator.smk_aek, &dh_smk, sizeof(sgx_key_128bit_t));
    memcpy(&session->initiator.shared_key, &shared_key, sizeof(sgx_ec256_dh_shared_t));
    // clear shared key and SMK
    memset_s(&shared_key, sizeof(sgx_ec256_dh_shared_t), 0, sizeof(sgx_ec256_dh_shared_t));
    memset_s(&dh_smk, sizeof(sgx_key_128bit_t), 0, sizeof(sgx_key_128bit_t));

    if(SGX_SUCCESS != sgx_ecc256_close_context(ecc_state))
    {
        // clear session
        memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
        // set error state
        session->initiator.state = SGX_DH_SESSION_STATE_ERROR;
        return SGX_ERROR_UNEXPECTED;
    }

    session->initiator.state = SGX_DH_SESSION_INITIATOR_WAIT_M3;
    return SGX_SUCCESS;

error:
    sgx_ecc256_close_context(ecc_state);

    // clear shared key and SMK
    memset_s(&shared_key, sizeof(sgx_ec256_dh_shared_t), 0, sizeof(sgx_ec256_dh_shared_t));
    memset_s(&dh_smk, sizeof(sgx_key_128bit_t), 0, sizeof(sgx_key_128bit_t));

    memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
    session->initiator.state = SGX_DH_SESSION_STATE_ERROR;

    // return selected error to upper layer
    INTERNAL_SGX_ERROR_CODE_CONVERTOR(se_ret)

    return se_ret;
}
```

这个函数是真的长啊。一开始检查了`msg1`, `msg2`, `session`, `role`的validity，然后创建了自己的keypair，根据`g_a`计算出来`shared_key`，销毁自己的`priv_key`；之后`shared_key`被`derive_key`做了一次key derivation, derive出来了`dh_smk`。根据LA的版本不同(v1 or v2)，`msg2`被generate (see below), 。最后，session的信息，包括`pub_key`, `peer_pub_key`, `smk_aek`, `shared_key`, `state`都被更新。

## LAv1 and LAv2

### Generator caller

```cpp
//sgx_LAv1_initiator_proc_msg1 processes M1 message, generates M2 message and makes update to the context of the session.
sgx_status_t sgx_LAv1_initiator_proc_msg1(const sgx_dh_msg1_t* msg1,
    sgx_dh_msg2_t* msg2, sgx_dh_session_t* sgx_dh_session)
{
    return dh_initiator_proc_msg1<dh_generate_message2>(msg1, msg2, sgx_dh_session);
}

// sgx_LAv2_initiator_proc_msg1() is a drop-in replacement of sgx_dh_initiator_proc_msg1()
sgx_status_t SGXAPI sgx_LAv2_initiator_proc_msg1(
    const sgx_dh_msg1_t* msg1, sgx_dh_msg2_t* msg2, sgx_dh_session_t* sgx_dh_session)
{
    return dh_initiator_proc_msg1<LAv2_generate_message2>(msg1, msg2, sgx_dh_session);
}
```

### `msg2` Generator

#### LAv2
```cpp
static sgx_status_t LAv2_generate_message2(
    const sgx_dh_msg1_t *msg1, const sgx_ec256_public_t *g_b,
    const sgx_key_128bit_t *dh_smk, sgx_dh_msg2_t *msg2)
{
    // No need to validate input parameters in an internal static function
    sgx_report_data_t rpt_data =  {{0}};
    bufcat(LAv2_proto_spec, *g_b).sha256(&rpt_data);

    sgx_target_info_t ti(msg1->target);
    sgx_report_t rpt;
    sgx_create_report(&ti, &rpt_data, &rpt);

    // Replace report_data with proto_spec
    rpt.body.report_data = LAv2_proto_spec.cast_to_report_data();

    // Put together LAv2 Message 2
    msg2->g_b = *g_b;
    msg2->report = rpt;
    return sgx_rijndael128_cmac_msg(dh_smk,
        reinterpret_cast<const uint8_t*>(&msg2->g_b), sizeof(msg2->g_b), &msg2->cmac);
}
```

这里report data(`rpt_data`)是把`LAv2_proto_spec`和`*g_b`concatenate起来之后再hash得到的，将会被塞入之后的report(`rpt`)之中。而这个report通过`sgx_create_report`里面调用`EREPORT`生成，`&ti`是Responder的target info，这样生成出来的report可以（**或许**是**只能**）被Responder解开并验证里面关于`rpt_data`的hash（这样可以防止MITM）。

之后，`msg2`里面囊括了`g_b`以及刚刚生成出来的`report`，这将被发给Responder。

#### LAv1

```cpp
static sgx_status_t dh_generate_message2(const sgx_dh_msg1_t *msg1,
                                         const sgx_ec256_public_t *g_b,
                                         const sgx_key_128bit_t *dh_smk,
                                         sgx_dh_msg2_t *msg2)
{
    sgx_report_t temp_report;
    sgx_report_data_t report_data;

    sgx_status_t se_ret;

    uint8_t msg_buf[MSG_BUF_LEN] = {0};
    uint8_t msg_hash[MSG_HASH_SZ] = {0};

    if(!msg1 || !g_b || !dh_smk || !msg2)
    {
        return SGX_ERROR_INVALID_PARAMETER;
    }

    memset(msg2, 0, sizeof(sgx_dh_msg2_t));
    memcpy(&msg2->g_b, g_b, sizeof(sgx_ec256_public_t));

    memcpy(msg_buf,
           &msg1->g_a,
           sizeof(sgx_ec256_public_t));
    memcpy(msg_buf + sizeof(sgx_ec256_public_t),
           &msg2->g_b,
           sizeof(sgx_ec256_public_t));

    se_ret = sgx_sha256_msg(msg_buf,
                            MSG_BUF_LEN,
                            (sgx_sha256_hash_t *)msg_hash);
    if(SGX_SUCCESS != se_ret)
    {
        return se_ret;
    }

    // Get REPORT with sha256(msg1->g_a | msg2->g_b) || kdf_id as user data
    // 2-byte little-endian KDF-ID: 0x0001 AES-CMAC Entropy Extraction and Key Derivation
    memset(&report_data, 0, sizeof(sgx_report_data_t));
    memcpy(&report_data, &msg_hash, sizeof(msg_hash));

    uint16_t *kdf_id = (uint16_t *)&report_data.d[sizeof(msg_hash)];
    *kdf_id = AES_CMAC_KDF_ID;

    // Generate Report targeted towards Session Responder
    se_ret = sgx_create_report(&msg1->target, &report_data, &temp_report);
    if(SGX_SUCCESS != se_ret)
    {
        return se_ret;
    }

    memcpy(&msg2->report, &temp_report, sizeof(sgx_report_t));

    //Calculate the MAC for Message 2
    se_ret = sgx_rijndael128_cmac_msg(dh_smk,
                                      (uint8_t *)(&msg2->report),
                                      sizeof(sgx_report_t),
                                      (sgx_cmac_128bit_tag_t *)msg2->cmac);
    if(SGX_SUCCESS != se_ret)
    {
        return se_ret;
    }

    return SGX_SUCCESS;
}
```
在v1中，`report_data`内存储的hash是`g_a || g_b`的hash。这个hash的尾部被`AES_CMAC_KDF_ID`进行填充。和v2一样，`cmac`会被计算出来装进`msg2`。

# `exchange_report_ocall`

和之前的`session_request_ocall`差不多，`msg2`被加上header封装成包发给Responder,然后再接收`msg3`。这里for sanity就不贴代码了。

# `exchange_report`

在Responder的server side代码中(`CPTask::run()`)，检查`header.type`为`FIFO_DH_MSG2`时调用处理函数`process_exchange_report`:
```cpp
            case FIFO_DH_MSG2:
            {
                // process ECDH message 2
                int clientfd = message->header.sockfd;
                SESSION_MSG2 * msg2 = NULL;
                msg2 = (SESSION_MSG2 *)message->msgbuf;

                if (process_exchange_report(clientfd, msg2) != 0)
                {
                    printf("failed to process exchange_report request.\n");
                    break;
                }
            }
```

## `process_exchange_report`

```cpp
int process_exchange_report(int clientfd, SESSION_MSG2 * msg2)
{
    uint32_t status = 0;
        sgx_status_t ret = SGX_SUCCESS;
    FIFO_MSG *response;
    SESSION_MSG3 * msg3;
    size_t msgsize;
    
    if (!msg2)
        return -1;
    
    msgsize = sizeof(FIFO_MSG_HEADER) + sizeof(SESSION_MSG3);
    response = (FIFO_MSG *)malloc(msgsize);
    if (!response)
    {
        printf("memory allocation failure\n");
        return -1;
    }
    memset(response, 0, msgsize);
    
    response->header.type = FIFO_DH_MSG3;
    response->header.size = sizeof(SESSION_MSG3);
    
    msg3 = (SESSION_MSG3 *)response->msgbuf;
    msg3->sessionid = msg2->sessionid; 

    // call responder enclave to process ECDH message 2 and generate message 3
    ret = exchange_report(e2_enclave_id, &status, &msg2->dh_msg2, &msg3->dh_msg3, msg2->sessionid);
    if (ret != SGX_SUCCESS)
    {
        printf("EnclaveResponse_exchange_report failure.\n");
        free(response);
        return -1;
    }

    // send ECDH message 3 to client
    if (send(clientfd, reinterpret_cast<char *>(response), static_cast<int>(msgsize), 0) == -1)
    {
        printf("server_send() failure.\n");
        free(response);
        return -1;
    }

    free(response);

    return 0;
}
```

## `exchange_report`

猜测是在EnclaveResponder内处理收到的`msg2`之后generate `msg3`的函数。

```cpp
//Verify Message 2, generate Message3 and exchange Message 3 with Source Enclave
extern "C" ATTESTATION_STATUS exchange_report(sgx_dh_msg2_t *dh_msg2,
                          sgx_dh_msg3_t *dh_msg3,
                          uint32_t session_id)
{

    sgx_key_128bit_t dh_aek;   // Session key
    dh_session_t *session_info;
    ATTESTATION_STATUS status = SUCCESS;
    sgx_dh_session_t sgx_dh_session;
    sgx_dh_session_enclave_identity_t initiator_identity;

    if(!dh_msg2 || !dh_msg3)
    {
        return INVALID_PARAMETER_ERROR;
    }

    memset(&dh_aek,0, sizeof(sgx_key_128bit_t));
    do
    {
        //Retrieve the session information for the corresponding source enclave id
        std::map<uint32_t, dh_session_t>::iterator it = g_dest_session_info_map.find(session_id);
        if(it != g_dest_session_info_map.end())
        {
            session_info = &it->second;
        }
        else
        {
            status = INVALID_SESSION;
            break;
        }

        if(session_info->status != IN_PROGRESS)
        {
            status = INVALID_SESSION;
            break;
        }

        memcpy(&sgx_dh_session, &session_info->in_progress.dh_session, sizeof(sgx_dh_session_t));

        dh_msg3->msg3_body.additional_prop_length = 0;
        //Process message 2 from source enclave and obtain message 3
        sgx_status_t se_ret = sgx_dh_responder_proc_msg2(dh_msg2,
                                                       dh_msg3,
                                                       &sgx_dh_session,
                                                       &dh_aek,
                                                       &initiator_identity);
        if(SGX_SUCCESS != se_ret)
        {
            status = se_ret;
            break;
        }

        //Verify source enclave's trust
          if(verify_peer_enclave_trust(&initiator_identity) != SUCCESS)
        {
            return INVALID_SESSION;
        }

        //save the session ID, status and initialize the session nonce
        session_info->session_id = session_id;
        session_info->status = ACTIVE;
        session_info->active.counter = 0;
        memcpy(session_info->active.AEK, &dh_aek, sizeof(sgx_key_128bit_t));
        memset(&dh_aek,0, sizeof(sgx_key_128bit_t));
        g_session_count++;
    }while(0);

    if(status != SUCCESS)
    {
        end_session(session_id);
    }

    return status;
}
```

这个代码有一个非常奇怪的用法`do {} while(0);`。实际上这个while循环只会跑一次，那么岂不是没有什么卵用？其实不然，这样它就可以通过`break`直接跳到`while(0);`下面而不需要使用丑陋的`goto`了。

一开始调用了`std::map<uint32_t, dh_session_t>::iterator`。实际上这玩意究竟是怎么用的我并不是很清楚，但是猜测是找到了一个现在接收到`msg2`的session。

然后调用`sgx_dh_responder_proc_msg2`处理`msg2`并生成`msg3`(see below)。

上一步如果成功，会接着验证peer enclave：`verify_peer_enclave_trust`。`initiator_identity`在处理`msg2`的过程中被生成。`verify_peer_enclave_trust`会check其`mr_signer`的值是否与自己(EnclaveResponder)所记录的一致。(**`mr_signer`是怎么得到的？**)

根据[官方的说法](https://software.intel.com/content/www/us/en/develop/blogs/introduction-to-intel-sgx-sealing.html)，MRSIGNER是：
> The second is the Signing Identity provided by an authority, which signs the enclave prior to distribution. This value is called MRSIGNER and will be the same for all enclaves signed with the same authority.

## `sgx_dh_responder_proc_msg2`

```cpp
/*Function name: sgx_dh_responder_proc_msg2
** parameter description
**@ [input] msg2: point to dh message 2 buffer generated by session initiator, and the buffer must be in enclave address space
**@ [output] msg3: point to dh message 3 buffer generated by session responder in this function, and the buffer must be in enclave address space
**@ [input/output] dh_session: point to dh session structure that is used during establishment, and the buffer must be in enclave address space
**@ [output] aek: AEK derived from shared key. the buffer must be in enclave address space.
**@ [output] initiator_identity: identity information of initiator including isv svn, isv product id, sgx attributes, mr signer, and mr enclave. the buffer must be in enclave address space.
*/
//sgx_dh_responder_proc_msg2 processes M2 message, generates M3 message, and returns the session key AEK.
sgx_status_t sgx_dh_responder_proc_msg2(const sgx_dh_msg2_t* msg2,
                                        sgx_dh_msg3_t* msg3,
                                        sgx_dh_session_t* sgx_dh_session,
                                        sgx_key_128bit_t* aek,
                                        sgx_dh_session_enclave_identity_t* initiator_identity)
{
    sgx_status_t se_ret;

    //
    // securely align shared key
    //
    //sgx_ec256_dh_shared_t shared_key;
    sgx::custom_alignment<sgx_ec256_dh_shared_t, 0, sizeof(sgx_ec256_dh_shared_t)> oshared_key;
    sgx_ec256_dh_shared_t& shared_key = oshared_key.v;
    //
    // securely align smk
    //
    //sgx_key_128bit_t dh_smk;
    sgx::custom_alignment_aligned<sgx_key_128bit_t, sizeof(sgx_key_128bit_t), 0, sizeof(sgx_key_128bit_t)> odh_smk;
    sgx_key_128bit_t& dh_smk = odh_smk.v;

    sgx_internal_dh_session_t* session = (sgx_internal_dh_session_t*)sgx_dh_session;
    bool is_LAv2 = false;

    // validate session
    if(!session ||
        0 == sgx_is_within_enclave(session, sizeof(sgx_internal_dh_session_t))) // session must be in enclave
    {
        return SGX_ERROR_INVALID_PARAMETER;
    }

    if(!msg3 ||
        msg3->msg3_body.additional_prop_length > (UINT_MAX - sizeof(sgx_dh_msg3_t)) || // check msg3 length overflow
        0 == sgx_is_within_enclave(msg3, (sizeof(sgx_dh_msg3_t)+msg3->msg3_body.additional_prop_length)) || // must be in enclave
        !msg2 ||
        0 == sgx_is_within_enclave(msg2, sizeof(sgx_dh_msg2_t)) || // must be in enclave
        !aek ||
        0 == sgx_is_within_enclave(aek, sizeof(sgx_key_128bit_t)) || // must be in enclave
        !initiator_identity ||
        0 == sgx_is_within_enclave(initiator_identity, sizeof(sgx_dh_session_enclave_identity_t)) || // must be in enclave
        SGX_DH_SESSION_RESPONDER != session->role)
    {
        // clear secret when encounter error
        memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
        session->responder.state = SGX_DH_SESSION_STATE_ERROR;
        return SGX_ERROR_INVALID_PARAMETER;
    }

    if(SGX_DH_SESSION_RESPONDER_WAIT_M2 != session->responder.state) // protocol state must be SGX_DH_SESSION_RESPONDER_WAIT_M2
    {
        // clear secret
        memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
        session->responder.state = SGX_DH_SESSION_STATE_ERROR;
        return SGX_ERROR_INVALID_STATE;
    }

    //create ECC context, and the ECC parameter is
    //NIST standard P-256 elliptic curve.
    sgx_ecc_state_handle_t ecc_state = NULL;
    se_ret = sgx_ecc256_open_context(&ecc_state);
    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }

    //generate shared key, which should be identical with enclave side,
    //from PSE private key and enclave public key
    se_ret = sgx_ecc256_compute_shared_dhkey((sgx_ec256_private_t *)&session->responder.prv_key,
                                            (sgx_ec256_public_t *)const_cast<sgx_ec256_public_t*>(&msg2->g_b),
                                            (sgx_ec256_dh_shared_t *)&shared_key,
                                             ecc_state);

    // For defense-in-depth purpose, responder clears its private key from its enclave memory, as it's not needed anymore.
    memset_s(&session->responder.prv_key, sizeof(sgx_ec256_private_t), 0, sizeof(sgx_ec256_private_t));

    if(se_ret != SGX_SUCCESS)
    {
        goto error;
    }


    //derive keys from session shared key
    se_ret = derive_key(&shared_key, "SMK", (uint32_t)(sizeof("SMK") -1), &dh_smk);
    if(se_ret != SGX_SUCCESS)
    {
        goto error;
    }
    // Verify message 2 from Session Initiator and also Session Initiator's identity
    se_ret = dh_verify_message2(msg2, &session->responder.pub_key, &dh_smk);
    if(SGX_SUCCESS != se_ret &&
       !(is_LAv2 = LAv2_verify_message2(msg2, &dh_smk)))
    {
        goto error;
    }

    initiator_identity->isv_svn = msg2->report.body.isv_svn;
    initiator_identity->isv_prod_id = msg2->report.body.isv_prod_id;
    memcpy(&initiator_identity->attributes, &msg2->report.body.attributes, sizeof(sgx_attributes_t));
    memcpy(&initiator_identity->mr_signer, &msg2->report.body.mr_signer, sizeof(sgx_measurement_t));
    memcpy(&initiator_identity->mr_enclave, &msg2->report.body.mr_enclave, sizeof(sgx_measurement_t));

    // Generate message 3 to send back to initiator
    se_ret = is_LAv2 ?
        LAv2_generate_message3(msg2, &session->responder.pub_key, &dh_smk, msg3) :
        dh_generate_message3(msg2, &session->responder.pub_key, &dh_smk, msg3,
                               msg3->msg3_body.additional_prop_length);
    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }

    // derive session key
    se_ret = derive_key(&shared_key, "AEK", (uint32_t)(sizeof("AEK") -1), aek);
    if(se_ret != SGX_SUCCESS)
    {
        goto error;
    }

    // clear secret
    memset_s(&shared_key, sizeof(sgx_ec256_dh_shared_t), 0, sizeof(sgx_ec256_dh_shared_t));
    memset_s(&dh_smk, sizeof(sgx_key_128bit_t), 0, sizeof(sgx_key_128bit_t));
    // clear session
    memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));

    se_ret = sgx_ecc256_close_context(ecc_state);
    if(SGX_SUCCESS != se_ret)
    {
        // set error state
        session->responder.state = SGX_DH_SESSION_STATE_ERROR;
        return SGX_ERROR_UNEXPECTED;
    }

    // set state
    session->responder.state = SGX_DH_SESSION_ACTIVE;

    return SGX_SUCCESS;

error:
    sgx_ecc256_close_context(ecc_state);
    // clear secret
    memset_s(&shared_key, sizeof(sgx_ec256_dh_shared_t), 0, sizeof(sgx_ec256_dh_shared_t));
    memset_s(&dh_smk, sizeof(sgx_key_128bit_t), 0, sizeof(sgx_key_128bit_t));
    memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
    // set error state
    session->responder.state = SGX_DH_SESSION_STATE_ERROR;
    // return selected error to upper layer
    if (se_ret != SGX_ERROR_OUT_OF_MEMORY &&
        se_ret != SGX_ERROR_KDF_MISMATCH)
    {
        se_ret = SGX_ERROR_UNEXPECTED;
    }
    return se_ret;
}
```

这里在C++中定义`dh_smk`和`shared_key`的时候用到了`&`，是C++中的一个feature。具体可以参考[这篇文章](https://www.geeksforgeeks.org/pointers-vs-references-cpp/)。与`sgx_dh_initiator_proc_msg1`
类似，进行输入变量的例行检查之后，计算出来`shared_key`，清空`responder.prv_key`。再derive key。

接下来便用v1(if not success, v2)版本的`dh_verify_message2`(`LAv2_verify_message2`)来检查`msg2`。(see below)。

`msg2`中关于platform的信息会被丢进`initiator_identity`中。之后便开始生成`msg3`(see below)。

之后便通过`shared_key`derive出AEK，session被正式建立。

## `msg2` verification

### `LAv2_verify_message2`

```cpp
static bool LAv2_verify_message2(const sgx_dh_msg2_t *msg2, const sgx_key_128bit_t *dh_smk)
{
    auto rpt = msg2->report;
    rpt.body.report_data = {{0}};
    bufcat(msg2->report.body.report_data, msg2->g_b).sha256(&rpt.body.report_data);
    if (SGX_SUCCESS != sgx_verify_report(&rpt))
        return false;

    if (SGX_SUCCESS != verify_cmac128(*dh_smk,
        reinterpret_cast<const uint8_t*>(&msg2->g_b), sizeof(msg2->g_b), msg2->cmac))
        return false;

    return memcmp(&msg2->report.body.report_data, &LAv2_proto_spec,
        offsetof(decltype(LAv2_proto_spec), rev)) == 0;
}
```
调用`sgx_verify_report`来verify `rpt`。先找到`report`中的`key_id`，然后通过`sgx_get_key`拿到对应的`report_key`。再用这个key对report做mac查看是否和原本report中的`mac`一致即完成验证。之后再验证`msg2`的cmac(*为什么还要验证msg2的呢？*)。

### `dh_verify_message2`

逻辑类似，不贴代码了。

## `msg3` generation

### `LAv2_generate_message3`

```cpp
static sgx_status_t LAv2_generate_message3(const sgx_dh_msg2_t *msg2,
    const sgx_ec256_public_t *A, const sgx_key_128bit_t *dh_smk, sgx_dh_msg3_t *msg3)
{
    sgx_status_t se_ret;
    auto& ps = LAv2_proto_spec.cast_from(msg2->report.body.report_data);

    sgx_target_info_t ti;
    if (SGX_SUCCESS != (se_ret =
        LAv2_proto_spec.make_target_info(ps, msg2->report, ti)))
        return se_ret;

    sgx_report_data_t rpt_data = {{0}};
    bufcat(*A, ps).sha256(&rpt_data);

    sgx_report_t rpt;
    sgx_create_report(&ti, &rpt_data, &rpt);
    msg3->msg3_body.report = rpt;

    sgx_cmac_state_handle_t cmac;
    if (SGX_SUCCESS != (se_ret = sgx_cmac128_init(dh_smk, &cmac)))
        return se_ret;

    sgx_cmac128_update(msg3->msg3_body.additional_prop,
        msg3->msg3_body.additional_prop_length, cmac);
    sgx_cmac128_update(reinterpret_cast<const uint8_t*>(A), sizeof(*A), cmac);
    sgx_cmac128_final(cmac, &msg3->cmac);
    sgx_cmac128_close(cmac);

    return SGX_SUCCESS;
}
```

类似的，和`msg2`生成的逻辑差不多，相关的信息被打包进了一个`report`。首先`ps`(proto spec)是从`msg2`的`report.body.report_data`中拿到的。然后`msg3`中的target info由`LAv2_proto_spec.make_target_info`准备好。注意这里的`make_target_info`和`msg1`中函数其实不是同一个。`_target_info_t`中的所有字段都在`_report_body_t`中出现，故而能够make出来。

### LAv1 `dh_generate_message3`

```cpp
static sgx_status_t dh_generate_message3(const sgx_dh_msg2_t *msg2,
                                         const sgx_ec256_public_t *g_a,
                                         const sgx_key_128bit_t *dh_smk,
                                         sgx_dh_msg3_t *msg3,
                                         uint32_t msg3_additional_prop_len)
{
    sgx_report_t temp_report;
    sgx_report_data_t report_data;
    sgx_status_t se_ret = SGX_SUCCESS;
    uint32_t maced_size;

    uint8_t msg_buf[MSG_BUF_LEN] = {0};
    uint8_t msg_hash[MSG_HASH_SZ] = {0};

    sgx_target_info_t target;

    if(!msg2 || !g_a || !dh_smk || !msg3)
    {
        return SGX_ERROR_INVALID_PARAMETER;
    }

    maced_size = static_cast<uint32_t>(sizeof(sgx_dh_msg3_body_t)) + msg3_additional_prop_len;

    memset(msg3, 0, sizeof(sgx_dh_msg3_t)); // Don't clear the additional property since the content of the property is provided by caller.

    memcpy(msg_buf, &msg2->g_b, sizeof(sgx_ec256_public_t));
    memcpy(msg_buf + sizeof(sgx_ec256_public_t), g_a, sizeof(sgx_ec256_public_t));

    se_ret = sgx_sha256_msg(msg_buf,
                            MSG_BUF_LEN,
                            (sgx_sha256_hash_t *)msg_hash);
    if(se_ret != SGX_SUCCESS)
    {
        return se_ret;
    }

    // Get REPORT with SHA256(g_b||g_a) as user data
    memset(&report_data, 0, sizeof(sgx_report_data_t));
    memcpy(&report_data, &msg_hash, sizeof(msg_hash));

    if (SGX_SUCCESS != (se_ret =
        LAv2_proto_spec.make_target_info(msg2->report, target)))
    {
        return se_ret;
    }

    // Generate Report targeted towards Session Initiator
    se_ret = sgx_create_report(&target, &report_data, &temp_report);
    if(se_ret != SGX_SUCCESS)
    {
        return se_ret;
    }

    memcpy(&msg3->msg3_body.report,
           &temp_report,
           sizeof(sgx_report_t));

    msg3->msg3_body.additional_prop_length = msg3_additional_prop_len;

    //Calculate the MAC for Message 3
    se_ret = sgx_rijndael128_cmac_msg(dh_smk,
                                      (uint8_t *)&msg3->msg3_body,
                                      maced_size,
                                      (sgx_cmac_128bit_tag_t *)msg3->cmac);
    if(se_ret != SGX_SUCCESS)
    {
        return se_ret;
    }

    return SGX_SUCCESS;
}
```

和LAv2类似，不多赘述。

# `sgx_dh_initiator_proc_msg3`

也是分为LAv1和LAv2。根据宏定义不同调用的函数分别为`sgx_LAv2_initiator_proc_msg3`和`sgx_LAv1_initiator_proc_msg3`。不论如何，区别仅仅在于调用的用来verify `msg3`的函数不同，其都使用了`dh_initiator_proc_msg3`。

## LAv2

```cpp
/*Function name: sgx_LAv2_initiator_proc_msg3
** parameter description
**@ [input] msg3: point to dh message 3 buffer generated by session responder, and the buffer must be in enclave address space
**@ [input/output] dh_session: point to dh session structure that is used during establishment, and the buffer must be in enclave address space
**@ [output] aek: AEK derived from shared key. the buffer must be in enclave address space.
**@ [output] responder_identity: identity information of responder including isv svn, isv product id, sgx attributes, mr signer, and mr enclave. the buffer must be in enclave address space.
*/
sgx_status_t sgx_LAv2_initiator_proc_msg3(const sgx_dh_msg3_t* msg3,
    sgx_dh_session_t* sgx_dh_session, sgx_key_128bit_t* aek,
    sgx_dh_session_enclave_identity_t* responder_identity)
{
    return dh_initiator_proc_msg3<LAv2_verify_message3>(
        msg3, sgx_dh_session, aek, responder_identity);
}
```

## LAv1

```cpp
/*Function name: sgx_LAv1_initiator_proc_msg3
** parameter description
**@ [input] msg3: point to dh message 3 buffer generated by session responder, and the buffer must be in enclave address space
**@ [input/output] dh_session: point to dh session structure that is used during establishment, and the buffer must be in enclave address space
**@ [output] aek: AEK derived from shared key. the buffer must be in enclave address space.
**@ [output] responder_identity: identity information of responder including isv svn, isv product id, sgx attributes, mr signer, and mr enclave. the buffer must be in enclave address space.
*/
sgx_status_t sgx_LAv1_initiator_proc_msg3(const sgx_dh_msg3_t* msg3,
    sgx_dh_session_t* sgx_dh_session, sgx_key_128bit_t* aek,
    sgx_dh_session_enclave_identity_t* responder_identity)
{
    return dh_initiator_proc_msg3<dh_verify_message3>(
        msg3, sgx_dh_session, aek, responder_identity);
}
```

### `dh_initiator_proc_msg3

```cpp
template <decltype(dh_verify_message3) ver_msg3>
static sgx_status_t dh_initiator_proc_msg3(const sgx_dh_msg3_t* msg3,
    sgx_dh_session_t* sgx_dh_session, sgx_key_128bit_t* aek,
    sgx_dh_session_enclave_identity_t* responder_identity)
{
    sgx_status_t se_ret;
    sgx_internal_dh_session_t* session = (sgx_internal_dh_session_t*)sgx_dh_session;

    // validate session
    if(!session ||
        0 == sgx_is_within_enclave(session, sizeof(sgx_internal_dh_session_t))) // session must be in enclave
    {
        return SGX_ERROR_INVALID_PARAMETER;
    }

    if(!msg3 ||
        msg3->msg3_body.additional_prop_length > (UINT_MAX - sizeof(sgx_dh_msg3_t)) || // check msg3 length overflow
        0 == sgx_is_within_enclave(msg3, (sizeof(sgx_dh_msg3_t)+msg3->msg3_body.additional_prop_length)) || // msg3 buffer must be in enclave
        !aek ||
        0 == sgx_is_within_enclave(aek, sizeof(sgx_key_128bit_t)) || // aek buffer must be in enclave
        !responder_identity ||
        0 == sgx_is_within_enclave(responder_identity, sizeof(sgx_dh_session_enclave_identity_t)) || // responder_identity buffer must be in enclave
        SGX_DH_SESSION_INITIATOR != session->role) // role must be SGX_DH_SESSION_INITIATOR
    {
        memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
        session->initiator.state = SGX_DH_SESSION_STATE_ERROR;
        return SGX_ERROR_INVALID_PARAMETER;
    }

    if(SGX_DH_SESSION_INITIATOR_WAIT_M3 != session->initiator.state) // protocol state must be SGX_DH_SESSION_INITIATOR_WAIT_M3
    {
        memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
        session->initiator.state = SGX_DH_SESSION_STATE_ERROR;
        return SGX_ERROR_INVALID_STATE;
    }

    se_ret = ver_msg3(msg3, &session->initiator.peer_pub_key,
        &session->initiator.pub_key, &session->initiator.smk_aek);
    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }

    // derive AEK
    se_ret = derive_key(&session->initiator.shared_key, "AEK", (uint32_t)(sizeof("AEK") -1), aek);
    if(SGX_SUCCESS != se_ret)
    {
        goto error;
    }

    // clear session
    memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
    session->initiator.state = SGX_DH_SESSION_ACTIVE;

    // copy the common fields between REPORT and the responder enclave identity
    memcpy(responder_identity, &msg3->msg3_body.report.body, sizeof(sgx_dh_session_enclave_identity_t));

    return SGX_SUCCESS;

error:
    memset_s(session, sizeof(sgx_internal_dh_session_t), 0, sizeof(sgx_internal_dh_session_t));
    session->initiator.state = SGX_DH_SESSION_STATE_ERROR;
    INTERNAL_SGX_ERROR_CODE_CONVERTOR(se_ret)
    return se_ret;
}
```

一开始还是照常的Enclave内的各种参数检查。这之后会根据LA的version来做一个`ver_msg3`。成功之后EnclaveInitiator side也会derive AEK，清除session信息。

## `LAv2_verify_message3`

```cpp
static sgx_status_t LAv2_verify_message3(const sgx_dh_msg3_t *msg3,
    const sgx_ec256_public_t *A, const sgx_ec256_public_t *, const sgx_key_128bit_t *dh_smk)
{
    auto rpt = msg3->msg3_body.report;
    rpt.body.report_data = {{0}};
    bufcat(*A, LAv2_proto_spec).sha256(&rpt.body.report_data);
    if (memcmp(&msg3->msg3_body.report.body.report_data,
            &rpt.body.report_data, sizeof(rpt.body.report_data)) ||
        SGX_SUCCESS != sgx_verify_report(&rpt))
        return SGX_ERROR_UNEXPECTED;

    sgx_cmac_state_handle_t cmac;
    if (SGX_SUCCESS != sgx_cmac128_init(dh_smk, &cmac))
        return SGX_ERROR_OUT_OF_MEMORY;

    sgx_cmac_128bit_tag_t tag;
    sgx_cmac128_update(msg3->msg3_body.additional_prop,
        msg3->msg3_body.additional_prop_length, cmac);
    sgx_cmac128_update(reinterpret_cast<const uint8_t*>(A), sizeof(*A), cmac);
    sgx_cmac128_final(cmac, &tag);
    sgx_cmac128_close(cmac);

    return consttime_memequal(&tag, msg3->cmac, sizeof(tag)) ?
        SGX_SUCCESS : SGX_ERROR_MAC_MISMATCH;
}
```

在LAv2内用来verify `msg3`的函数。与之前的EnclaveResponder在verify `msg2`时几乎做的是同样的事情：`sgx_verify_report`，检查`cmac`一致性。

# After story

`EnclaveInitiator`在处理完`msg3`之后，和`EnclaveResponder`一样，也要验证一下(`verify_peer_enclave_trust`) `responder_identity`。完成之后整个session的AEK被建立起来，session变成active。

这之后便可以开始使用这个session的密钥(AEK)来交换消息了。消息交互的逻辑似乎比较简单，如果有必要的话我说不定会再写一篇。

# 总结

这次仔细研读SGX SDK里面的代码可以看出来其对于安全的考量还是比较多的，写的也相当规范。

总体而言，SDK里面的Sample Code是真的很复杂，SDK里面的过程也很繁琐。但是逻辑还是相对比较清晰的，基本上SDK里面的每一个函数都要经历：对参数的检查（不为空，全部在enclave里面），对于输出buffer的创建和清零。这之后才会经历正常的代码处理逻辑。