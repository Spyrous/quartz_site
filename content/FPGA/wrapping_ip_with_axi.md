---
title: "Wrapping an IP with an AXI interface in Vivado"
date: 2021-11-15T11:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: ["AXI", "Xilinx", "Vivado", "Vitis", "FPGA"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
# description: "Desc Text."
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
# editPost:
#     URL: "https://github.com/<path_to_repo>/content"
#     Text: "Suggest Changes" # edit text
#     appendFilePath: true # to append file path to Edit link
---

## Introduction

In this tutorial we will design a simple adder IP and use Xilinx Vivado AXI wrapper generator to allow our ARM cores to read/write to our IP through the AXI interface.

The high-level design will be the one in the image below. We will have 2 IPs. One is our custom made IP wrapped with AXI interface and an AXI BRAM controller just for demonstration purposes.

## Tools Used

The following tools where used to create this tutorial:
* ZyBo FPGA
* Vivado 2021.1
* Ubuntu 20.04 LTS


![Final design high-level sketch](images/axi_custom.png#center)

## Hardware Part

To begin start Xilinx Vivado and create a new design.

![Create block design](images/create_block_design.png#center)

![Add zynq IP](images/add_ips_zynq.png#center)

![Add BRAM IP](images/add_ips_axi_bram.png#center)

![Run auto block and connection](images/run_auto_block_and_connection.png#center)

```Tools --> Create and Package New IP ...```

![Create and Package New IP...](images/create_axi_component.png#center)

Then click ```Next >```

On the next screen select ```Create a new AXI4 peripheral``` and then click ```Next >```

Name your IP as you want. Me I named it axi_adder. Then click ```Next >```

On the tab with the interface of the AXI leave everything as is and just click ```Next >```

After in the final tab select ```Edit IP``` and click ```Finish```.

This will open your AXI wrapper IP in a new Vivado window.
Inside this AXI wrapper we need to add our own IP.
To add an IP click on the '+' symbol. 

![Add sources](images/asdf.png#center)

Then click ```Create File```. In the window that pops up put us ```File Type``` SystemVerilog, ```File Name``` adder
and ```File Location``` anywhere you want preferably in the same folder as the IP. Click ```OK``` and then ```Finish```.

A new window will popup called ```Define Module``` to help you create the header of your IP. You can ignore it since
you will copy paste the code below. Click ```OK``` to generate the file.

In your hierarchy now you should have the adder file.

![Adder file hierarchy](images/adder_file_hierarchy.png#center)

In the adder.sv file copy and paste the following code:

```verilog
module adder(
    input  logic [31:0] num1,
    input  logic [31:0] num2,
    output logic [31:0] sum
);

assign sum = num1 + num2;

endmodule
```

Now we need to connect the 2 inputs to the 2 out of the 4 registers provided from the AXI slave interface and the output to one of the 2 remaining. To do this open the AXI slave file axi_adder_v1_0_S00_AXI.v and change the following parts:

Between lines 103 and 110 replace the 'reg' of slv_reg2 with 'wire' as it will be our output register.

```verilog {7}
//----------------------------------------------
//-- Signals for user logic register space example
//------------------------------------------------
//-- Number of Slave Registers 4
reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg0;
reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg1;
wire [C_S_AXI_DATA_WIDTH-1:0]	slv_reg2; // Turned to wire for the adder output
reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg3;
```

From the process that writes to all slv_reg registers (lines 219-269) remove the slv_reg2 as it will be written from our IP.

```verilog {7,33,45}
always @( posedge S_AXI_ACLK )
begin
    if ( S_AXI_ARESETN == 1'b0 )
    begin
        slv_reg0 <= 0;
        slv_reg1 <= 0;
//	      slv_reg2 <= 0;
        slv_reg3 <= 0;
    end 
    else begin
    if (slv_reg_wren)
    begin
    case ( axi_awaddr[ADDR_LSB+OPT_MEM_ADDR_BITS:ADDR_LSB] )
        2'h0:
        for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
            if ( S_AXI_WSTRB[byte_index] == 1 ) begin
            // Respective byte enables are asserted as per write strobes 
            // Slave register 0
            slv_reg0[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
            end  
        2'h1:
        for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
            if ( S_AXI_WSTRB[byte_index] == 1 ) begin
            // Respective byte enables are asserted as per write strobes 
            // Slave register 1
            slv_reg1[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
            end  
        2'h2:
        for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
            if ( S_AXI_WSTRB[byte_index] == 1 ) begin
            // Respective byte enables are asserted as per write strobes 
            // Slave register 2
//	                slv_reg2[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
            end  
        2'h3:
        for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
            if ( S_AXI_WSTRB[byte_index] == 1 ) begin
            // Respective byte enables are asserted as per write strobes 
            // Slave register 3
            slv_reg3[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
            end  
        default : begin
                    slv_reg0 <= slv_reg0;
                    slv_reg1 <= slv_reg1;
//	                      slv_reg2 <= slv_reg2;
                    slv_reg3 <= slv_reg3;
                end
    endcase
    end
    end
end
```

And just before the **endmodule** keyword there are 2 comments saying 'Add user logic here'. Between those comments add the following code:

```verilog
// Add user logic here
adder adder_i (
    .num1 (slv_reg0),
    .num2 (slv_reg1),
    .sum  (slv_reg2)
);
// User logic ends
```

This will connect our IP's 2 inputs to 2 of the slv_reg that we can read/write from the CPU and connect the output to slv_reg2 which we will be able to read-only.

**WARNING: It is adviced to try a synthesis run before closing the IP to make sure there was no mistake during the code editing part.**

After that after merging the changes and you are asked if you want to close the project click OK.

Back on our design now that we created our IP we need to add it on our design. Just open the block design by clicking ```Open Block Design``` on the left and
right click in the schematic and click ```Add IP...``` and search for ```axi_adder``` and add it in the design. Then on top you will again get the do automatic connection suggestion
from Vivado. Click on it and let Vivado create a connection to our axi_adder automatically from the interconnect.

In the end your schematic diagram should look like the one on the image below:

![Final diagram](images/final_diagram.png#center)

Then you need to create an HDL wrapper and generate the bitstream.

![Create HDL Wrapper](images/create_hdl_wrapper.png#center)

After this is done export the hardware with the bitstream.

![Export XSA file](images/export_xsa.png#center)

On the first window click ```Next >``` and then on the next window from the options check the ```Include bitstream``` and then click ```Next >```. On the next window make sure you remember the path where you save the .xsa file as we will source it in the Vitis tool to create our software.

Then open the Vitis toolchain by clicking from the top tabs
```Tools --> Launch Vitis IDE``` and then click Launch on the next popup window to start the software part of the design.



## Software Part

After Vitis SDK loads we start by creating a new application.

![Create new application](images/create_new_application.png#center)

On the first popup welcome window click ```Next >``` and then on the second window
choose to create a new platform with the XSA file created from Vivado.

![Create XSA platform](images/create_xsa_platform.png#center)

Then on the next window called Application Project Details give a name to your project for example axi_adder_test and leave the remaining choices as they are.
Next window called Domain click ```Next >``` and on the final window called Templates
choose ```Hello World``` and then click ```Finish```.

We create this application to initiate our CPU cores to a working state and allow us to access the FPGA by lowering the level shifters.

To test our IP through software I put 2 ways. The first one which I consider the easiest is with XSCT. The second one is writing a C application and watching through a UART terminal the output.

### Testing the hardware with XSCT

Now we need to first build the project and then run it.
To build right-click the C application and run ```Build Project``` and wait for some
seconds for it to build.

![Build Software Project](images/build_project.png#center)

After the application has been build connect to your FPGA board and power it on.

Run the application on the FPGA. This will program the FPGA and run the hello world test on it.

![Run application](images/run_application.png#center)

Now we should be able to use the XSCT console to access the FPGA part through the ARM
cores. To do this you can run the commands below on the XSCT console.

![Testing the adder](images/testing_the_adder.png#center)

Here I use the first 2 registers of the design as input numbers and the third one as output of the sum of the addition. The addresses used were given from Vivado. You can find them in the address editor in Vivado tool in your schematic viewer.

![Address editor](images/address_editor.png#center)


### Testing the hardware with a C application

We can also do it using a C program and a UART terminal to read the results.
For a UART terminal in this example I used Putty and I opened it using ```sudo putty```.

![Putty config](images/putty_config.png#center)

To find which port to listen to for UART data I did a ```dmesg``` on my terminal and when I powered-on my FPGA I saw which ttyUSB port it used. 

![Putty listen port](images/putty_listen_port.png#center)

For the UART baud rate (or the field Speed in Putty) you can look inside the comments in the helloworld.c file. It is 115200.

![helloworld file](images/helloworld_cfile.png#center)


Now copy this C code and replace the int main() program inside the helloworld.c file.

```c
#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"


int main()
{
    init_platform();
    uint32_t *axi_adder_ptr = (uint32_t *) 0x43C00000;
    *axi_adder_ptr = 0x11223344;
    *(axi_adder_ptr + 1) = 0x11223344;

    print("Hello World\n\r");
    printf("The axi adder result is 0x%x", *(axi_adder_ptr + 2));
    cleanup_platform();
    return 0;
}
```

Build the project and run again with your Putty session open and watch the output of the print.

![Putty terminal](images/putty_terminal.png#center)

## Conclusion

This concludes the tutorial of designing your hardware IP wrapping it with AXI interface in Vivado and testing its functionality through the Vitis SDK and Zynq cores.
