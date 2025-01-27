# Domain-Specific Accelerator (DSA)
## Abstract
This project implements a **Domain-Specific Accelerator (DSA)** on the **Aquila processor** using **Memory-Mapped I/O (MMIO)**. By leveraging Vivado to add custom IPs or modify circuits on Aquila, the execution time of a CNN model was significantly reduced. The final results achieved a **20x speedup** in performance.
## Basic Concept of DSA
### MMIO
Through **Memory-Mapped I/O (MMIO)**, communication between the software and the DSA on the Aquila processor is enabled. According to the comments in `soc_top.v` under the device address decoder, the memory address range `0xC400_0000` to `0xC4FF_FFFF` is reserved for the DSA. This means that when load/store instructions access this memory address range, `soc_top.v` uses the signal `dsa_sel` to select the DSA device. The data to be stored is sent to the DSA, and the data output from the DSA is retrieved through the load operation.
### Floating Point IP
The floating-point IP was configured with the **Fused-Multiply-Add** option and set to a **non-blocking** mode. This approach simplifies the control signals, avoiding the need to handle more complex logic. 

Initially, the latency was set to `0`, but this caused the computations to fail. Eventually, the latency was configured to `2`, which resolved the issue. To maintain clarity, the IP was designed to operate in a manner where a `data_valid` signal is triggered only after all three input data values have been read. At this point, the three inputs are sent to the IP.

Once the result is valid, it is temporarily stored in a register to preserve the computation's output. Only after this step is completed will the next set of three input data values be read, ensuring that the input and output of the IP are handled in consistent groups.
### Software Sructure of MMIO and DSA
```
for (uint64_t c = 0; c < entry->base.in_size; c++){
    *((float volatile *)0xC4000000) = W[i*entry->base.in_size_ + c];
    *((float volatile *)0xC4000004) = in[c];
    *((float volatile *)0xC4000008) = a[i];
    a[i] = *((float volatile *)0xC4000010)
}
```
## Advance Implementation
### Pooling Layer
At the memory address `0xC400_0024`, the value `dxmax * dymax` is read and assigned to `pool_img_size_i`. A counter is then used to determine the required indices of the `in[]` array, where the index is precomputed in software. Once all data is read, the state transitions back to `P_IDLE`. 

Next, the address `0xC400_002C` is set as the calculating trigger. Upon being triggered, the DSA utilizes a floating-point add IP to sum all the values stored in `in[]`. After the summation, the result is multiplied by `scale_factor_` using the multiply IP. For the average pooling operation, `scale_factor_` is directly set to `0.25`, which is represented in floating-point format as `0x3e800000`. 

Once the computation is complete, the trigger is set to `0`, exiting the busy waiting state in the while loop. Finally, the result is stored at address `0xC400_0030`, completing one iteration of the operation.
### Converlution Layer
1. The two main data components, the input image and weights, are preloaded into the circuit.  
2. Registers are used to store these data, simplifying the I/O process. The RAM style is set to "block" to ensure that the registers storing these data are synthesized as block RAM instead of using LUTs.  
3. The `conv_3d()` circuit is controlled using an FSM. After loading the necessary parameters, the FSM initializes the output register at address `0xC430_0000`. Once initialization is complete, it transitions to the `load weight` and `load image data` states to load the corresponding data. During this time, the software enters a busy waiting state until the FSM returns to `S_IDLE`. Since the weight width is 5, 25 inner product operations are performed per round. After each round, the FSM transitions to `S_STORE` to save the results to the output register. Once all inner product operations are completed, the FSM returns to `S_IDLE`, exiting the busy waiting state.  
4. Certain unnecessary calculations were simplified, such as directly calculating the result when padding is `0` and stride is `1`, to reduce the computation load.
## Result and Discussion
### Current Performance
Execution time reduced: 21354 ms → **1067** ms (**20.01x** speedup).
### IP Latency amd WNS
| Version       | Baseline | Normal | Change Latency |
|:---------------:|:----------:|:--------:|:----------------:|
| **Spend Time (ms)** | 21354    | 1067   | 990            |
| **Speedup Rate**    | 1        | 20.01  | 21.60          |
| **Clk Rate**        | 50 MHz   | 50 MHz | 50 MHz         |
| **IP Latency**      | 2        | 2      | 1              |
| **WNS**            | > 0      | > 0    | -12.59         |

Reducing latency successfully accelerates the operation of the DSA while maintaining an accuracy rate of 95%. However, after setting the IP latency to `1`, the Setup Time's **WNS (Worst Negative Slack)** dropped to `-12.59`. This indicates that the current operation cannot be completed within the designated cycle. While the accuracy rate remains unchanged, some operations may have actually failed under these conditions.
### Resource Utilzation
| Version           | Baseline | Normal | Optimize |
|:-------------------:|:----------:|:--------:|:----------:|
| **Spend Time (ms)**   | 21354    | 1067   | 1183     |
| **Speedup Rate**      | 1        | 20.01  | 18.05    |
| **Slice LUTs**        | N/A      | 8568   | 6837     |
| **Slice**             | N/A      | 2621   | 2064     |
| **LUT as Logic**      | N/A      | 3660   | 3337     |
| **LUT as Memory**     | N/A      | 4908   | 3500     |
| **Block RAM**         | N/A      | 5      | 5        |

## Future Work
1. Integration
    Converting the entire CNN model to hardware execution significantly improves efficiency and reduces the number of I/O operations.
2. Quantization
    By converting the image and weight data into 8-bit integer values and using a mixed-precision approach, it is possible to combine four 8-bit integers into a single 32-bit value while maintaining accuracy. This approach significantly reduces the number of I/O transfers. Additionally, the DSA can perform computations using integer operations instead of relying heavily on floating-point IPs, simplifying the design of the DSA.
3. Parallel Operations
    By precomputing the indices and using multiple IPs to process the data simultaneously, more computations can be executed within a single clock cycle, thereby improving the overall speed.
