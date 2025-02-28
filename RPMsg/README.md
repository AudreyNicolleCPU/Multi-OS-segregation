# RPMsg
## What is RPMsg ? 

RPMsg stands for Remote processsor Messaging. If you want to learn more about its protocol, its communication flow and Linux software implementation, please see the following documents : 
- Theoritical functionnement : https://openamp.readthedocs.io/en/latest/protocol_details/rpmsg.html.
- Linux implementation : https://docs.kernel.org/staging/rpmsg.html.

Here a short summary of it : 

The RPMsg protocol is used for communication between cores in asymmetric multiprocessor systems, utilizing shared memory for data exchange. It consists of three layers: the physical layer using shared memory, the media access layer (VirtIO for example) for efficient data transfer, and the transport layer (RPMsg) that manages message communication. RPMsg messages contain headers for routing data between cores, and the system supports multiple communication channels and endpoints for efficient message delivery. The protocol enables core-to-core communication without requiring additional hardware synchronization.

RPMsg communication involves two cores, with one acting as the Master and the other as the Remote. The Master allocates buffers for message transmission and enqueues data to the “avail” ring buffer, while the Remote processes the message and returns it to the “used” ring buffer. When the Remote sends messages, the roles of the buffers are reversed. The Master core controls memory management and may throttle communication by not filling the buffers. Interrupts are optional and controlled by flags in the buffers.

To implement RPMsg we used iMX95 board from NXP with Linux on the A55 cores and Zephyr (FreeRTOS to test with NXP given examples) on M7 core. 

## RPMsg on Linux side

### Node structure in Linux Device tree

```js
/{
  cm7: imx95-cm7 {
        compatible = "fsl,imx95-cm7";
        mbox-names = "tx", "rx", "rxdb";
        mboxes = <&mu7 0 1
              &mu7 1 1
              &mu7 3 1>;
        memory-region = <&vdevbuffer>, <&vdev0vring0>, <&vdev0vring1>,
                <&vdev1vring0>, <&vdev1vring1>, <&rsc_table>;
        fsl,startup-delay-ms = <50>;
        status = "okay";
    };
};
```

#### What was the "error" in the node that was not making the example working 

While running FreeRTOS examples, when trying to load the module, Linux was giving this error about remoteproc : 

```bash
root@imx95evk:~# modprobe imx_rpmsg_pingpong
  [ 3522.273511] imx_rpmsg_tty virtio1.rpmsg-openamp-demo-channel.-1.30: new channel: 0x400 -> 0x1e!
  [ 3522.282550] Install rpmsg tty driver!
  [ 3522.389090] imx-rproc imx95-cm7: imx_rproc_kick: failed (3, err:-62)
```
This led us to believe that remoteproc and RPMsg module were linked. We found out that the remoteproc module was indeed loading some RPMSg configurations but RPMsg can be loaded directly without the help of remoteproc, please see section 3.2 of https://wiki.st.com/stm32mpu/wiki/Linux_RPMsg_framework_overview. 

For the time being, remoteproc is not working on our side, so we decided to directly use the imx95 RPMsg module by changing the compatible property of the node. Since the board we are using is still in pre-production, RPMsg does not seem to have been fully implemented, and we could not find the compatible property for it. Therefore, we used the one from a more recent board similar to ours, which gives the following node declaration:

```js
/{
  cm7: imx95-cm7 {
        compatible = "fsl,imx8qm-rpmsg";
        mbox-names = "tx", "rx", "rxdb";
        mboxes = <&mu7 0 1
              &mu7 1 1
              &mu7 3 1>;
        memory-region = <&vdevbuffer>, <&vdev0vring0>, <&vdev0vring1>,
                <&vdev1vring0>, <&vdev1vring1>, <&rsc_table>;
        fsl,startup-delay-ms = <50>;
        status = "okay";
    };
};
```
With this node declaration, the RPMsg protocol is working between Linux and the M7 core. Now, let's see what properties and the better node structure to use, depending on what we want to do for now.

#### Mailbox Message Unit properties
```js
/{
  cm7: imx95-cm7 {
      ...
      mbox-names = "tx", "rx", "rxdb";
      mboxes = <&mu7 0 1
            &mu7 1 1
            &mu7 3 1>;
      ...
    };
};
```
*mboxes* property is here to link a Message Unit (MU) memory space to the RPMsg node we would be using with the mailbox Linux module. Here we are using *mu7* unit which is defined in imx95.dtsi file : 
```js
  mu7: mailbox@42430000 {
      compatible = "fsl,imx95-mu", "fsl,imx8ulp-mu";
      reg = <0x42430000 0x10000>;
      interrupts = <GIC_SPI 234 IRQ_TYPE_LEVEL_HIGH>;
      clocks = <&scmi_clk IMX95_CLK_BUSWAKEUP>;
      #mbox-cells = <2>;
      status = "disabled";
  };
```
**Why disables ? Try to remove "fsl,imx8ulp-mu" to see if it's still working**
This MU is definded in AIPS (Advanced Independent Peripheral Subsystem) peripheral bridge memory region which every core as access to. 
This MU support 4 type of unidirectional channels, each type has 4 channels. A total of 16 channels. Following types are supported:
- 0 - TX channel.
- 1 - RX channel.
- 2 - TX doorbell channel.
- 3 - RX doorbell channel.

| Feature | TX/RX Channels | TX/RX Doorbell Channels |
| :----  | :----: | :----:  |
| Purpose | Data transmission and reception | Signaling transmit/receive action |
| Data Register	 |32-bit transmit/receive register | No dedicated register |
| Acknowledge Support |Yes | No |		
| Use Case |Sending/receiving data with acknowledgment | Triggering transmit/receive event |			

Looking back to *cm7* RPMsg node declaration, in *&mu7 0 1*, the first number, here 0, is what kind of channel is declared, here TX and 1 is to say that over the 4 possible TX channel it is the first one that is used. The property *mobo-names* gives identifier to the Mailbox defined. 

For more information about it, please see : 
- https://www.nxp.com/docs/en/application-note/AN2741.pdf
- https://patchwork.kernel.org/project/linux-arm-kernel/patch/20190531143320.8895-2-sudeep.holla@arm.com/
- https://patchwork.kernel.org/project/linux-arm-kernel/patch/20180731141146.10788-3-o.rempel@pengutronix.de/
- https://docs.kernel.org/driver-api/mailbox.html

#### Memory-region property

#### Bare-bones node structure
```js
/{
  cm7: imx95-cm7 {
        compatible = "fsl,imx8qm-rpmsg";
        mbox-names = "tx", "rx";
        mboxes = <&mu7 0 1
              &mu7 1 1>;
        memory-region = <&vdevbuffer>;
        fsl,startup-delay-ms = <50>;
        status = "okay";
    };
};
```
## RPMsg on Zephyr Side

## RPMsg on System Monitor (SM) side

## Examples

### FReeRTOS examples

You can find FreeRTOS examples in *SDK_24_12_00_IMX95LPD5EVK-19/boards/imx95lpd5evk19/multicore_examples/*.
Please used this tutorial to help running them : https://developer.toradex.com/software/cortex-m/cortexm-rpmsg-guide/.

### RPMsg example
