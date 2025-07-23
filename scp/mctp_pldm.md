# MCTP和PLDM调用流程分析





## 模块设计框图



```mermaid
flowchart RL
	subgraph DRIVER
	direction RL
	MCTP_SERIAL
	MCTP_PCC
	end
	MCTP_FW-->DRIVER
	
	PLDM-->MCTP-->MCTP_FW

```



## 从应用层发送消息的物理层的时序图

1. PLDM_FW模块，或其它应用层，调用mctp_fw_api.mctp_fw_receive_from_app_layer()接口，将协议层消息下发到MCTP传输层
2. MCTP_FW模块，添加MCTP消息头，调用process_mctp_fw_tx，具体调用mctp_api->mctp_message_tx，将消息发给物理层（Serial或PCC）
3. 如果物理层走MCTP_Serial，则调用MCTP_SERIAL模块注册的tx接口：mod_mctp_serial_tx_fn
4. 如果物理层走MCTP_PCC，则调用MCTP_PCC模块注册的tx接口：mod_mctp_pcc_tx_fn

```mermaid
sequenceDiagram
	autonumber
    participant PLDM_FW
    participant MCTP_FW
    participant MCTP_SERIAL
    participant MCTP_PCC
    participant TRANSPORT
    PLDM_FW->>+MCTP_FW: send_pldm_packet()
    MCTP_FW->>MCTP_FW: mctp_fw_receive_from_app_layer
    MCTP_FW->>+MCTP_SERIAL: mctp_message_tx()
    MCTP_FW->>+MCTP_PCC: mctp_message_tx()
    loop send mctp_packet
    	MCTP_SERIAL->>MCTP_SERIAL: fwk_io_putch
    end
    MCTP_PCC->>MCTP_PCC: mod_mctp_pcc_tx_fn
    MCTP_PCC->>+TRANSPORT: transport_api->transmit
    TRANSPORT->>TRANSPORT: transmit message
    TRANSPORT->>MCTP_PCC: return status
    deactivate TRANSPORT
    MCTP_PCC->>MCTP_FW: return status
    deactivate MCTP_PCC
    MCTP_SERIAL->>MCTP_FW: return len
    deactivate MCTP_SERIAL
    MCTP_FW->>-MCTP_FW: free buffer
```



## 从物理层收到消息并交给协议栈处理流程图

1. 物理层为Serial，则在mod_mctp_serial.c模块文件里，用到了TIMER定时器，周期性的轮询串口，将接收到的字符上传给MCTP核心层
2. 物理层为PCC，待分析
3. MCTP核心层，负责解析从串口接收的原始字节流，并组装成完整的MCTP数据包，然后调用mctp_bus_rx将数据包传递给MCTP核心协议栈
4. MCTP核心层通过mctp_set_rx_all注册的回调接口（mod_mctp_fw.c里，通过mctp_api->mctp_set_rx_all，注册了mctp_fw_receive_from_mctp），进一步将数据传递给应用层（如PLDM）
5. PLDM应用层，通过接口pldm_fw_receive_from_transport_layer，处理MCTP核心层送上来的报文。

```mermaid
sequenceDiagram
	autonumber
    participant MCTP_SERIAL
    participant MCTP
    participant MCTP_FW
    participant PLDM_FW
    participant PLDM
 
    loop get char from pl011
    	MCTP_SERIAL->>MCTP_SERIAL: fwk_io_getch
    end
    MCTP_SERIAL->>+MCTP: mctp_serial_rx
    activate MCTP
    MCTP->>MCTP: mctp_rx_consume
    MCTP->>MCTP: mctp_bus_rx
    MCTP->>MCTP: mctp_rx
    MCTP->>+MCTP_FW: mctp_fw_receive_from_mctp
    deactivate MCTP 
    alt is MCTP message
    	MCTP_FW->>MCTP_FW:process_mctp_fw_tx
    	MCTP_FW->>MCTP: mctp_message_tx
    else is pldm message
    	MCTP_FW->>PLDM_FW: pldm_fw_receive_from_transport_layer    	
    	activate PLDM_FW
    end
    deactivate MCTP_FW
    
    alt is pldm_request_message
    	PLDM_FW->>PLDM_FW: process_pldm_packet
    	PLDM_FW->>+MCTP_FW: send_pldm_packet
    	MCTP_FW->>-MCTP_FW: mctp_fw_receive_from_app_layer
    	MCTP_FW->>+MCTP: mctp_api->mctp_message_tx
    	MCTP->>-MCTP: mctp_packet_tx
    	MCTP->>+MCTP_SERIAL:mod_mctp_serial_tx_fn
    	MCTP_SERIAL->>-MCTP_SERIAL: fwk_io_putch
   	else is pldm_response
    	PLDM_FW->>PLDM_FW:process_pldm_packet
    	PLDM_FW->>+PLDM: pldm_base_api->encode_cc_only_resp
    	PLDM->-PLDM: pldm_base_api
	    deactivate PLDM_FW
   	end   	

```

