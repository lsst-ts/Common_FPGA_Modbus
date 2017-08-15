Usage for each port:
1. Generate a FIFO for Rx data. Specify a data type of U16. Specify the write to never arbitrate.
2. Generate a FIFO for Rx health and status. Specify a data type of HealthAndStatusUpdate.ctl. Specify never arbitrate for write and read.
3. Generate a FIFO for Tx data. Specify a data type of U16. Specify never arbitrate for read.
4. Generate a FIFO for internal Tx usage. Specify a data type of ModbusTxInstruction.ctl. Specify never arbitrate for write and read. 
5. Generate a FIFO for Tx health and status. Specify a data type of HealthAndStatusUpdate.ctl. Specify never arbitrate for write and read.
6. Generate a register for IRQ control. Specify a boolean data type. Specify never arbitrate for write.
7. Generate a register for wait for rx frame. Specify a boolean data type. Specify never arbitrate for write.
8. Generate a register for the software controlled trigger. Specify a boolean data type. Specify never arbitrate for write. You can use a single trigger for controlling multiple ports so this is an optional step if a register has already been generated.
9. Generate a FIFO for Tx timestamps. Specify a data type of U64. Specify never arbitrate for write and read.
10. Drop the FPGAModbus/ModbusPort.vi onto your main VI. Setup the port settings using the resources defined above. Review the comments in ModbusPort.vi for more information.
11. Drop the FPGAHealthAndStatus/HealthAndStatusFIFOCopy.vi. Set the internal health and status FIFO. Set the source as the FIFO generated in step 2.
12. Drop the FPGAHealthAndStatus/HealthAndStatusFIFOCopy.vi. Set the internal health and status FIFO. Set the source as the FIFO generated in step 5.