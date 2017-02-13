Usage for each port:
1. Generate a target to host - DMA FIFO for Rx data. Specify a data type of U16. Specify the write to never arbitrate.
2. Generate a target-scoped FIFO for Rx health and status. Specify a data type of HealthAndStatusUpdate.ctl. Specify never arbitrate for write and read.
3. Generate a host to target - DMA FIFO for Tx data. Specify a data type of U16. Specify never arbitrate for read.
4. Generate a target-scoped FIFO for internal Tx usage. Specify a data type of ModbusTxInstruction.ctl. Specify never arbitrate for write and read. 
5. Generate a target-scoped FIFO for Tx health and status. Specify a data type of HealthAndStatusUpdate.ctl. Specify never arbitrate for write and read.
6. Generate a register for IRQ control. Specify a boolean data type. Specify never arbitrate for write.
7. Generate a register for wait for rx frame. Specify a boolean data type. Specify never arbitrate for write.
8. Generate a register for the software controlled trigger. Specify a boolean data type. Specify never arbitrate for write. You can use a single trigger for controlling multiple ports so this is an optional step if a register has already been generated.
8. Drop the ModbusPort.vi onto your main VI. Setup the port settings using the resources defined above. Review the comments in ModbusPort.vi for more information.