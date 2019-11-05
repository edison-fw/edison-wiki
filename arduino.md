# Edison/Arduino

## Edison/Arduino expansion board
The expansion board has known (design) flaws:
  * Discrete pin control has an (unknown) issue when I2C GPIO expanders, powered by V_SHLD_IO, are simple disappeared. This requires the cold power off / on cycle in order to recovery.
  * The design of TRI_STATE_ALL doesn't allow us to set that pin reliably. (In the correct design the TRI_STATE_ALL must be a pin of the GPIO which always present in the system, like the main one on SoC).

That is, Edison/Arduino expansion board is not recommended for production devices. If you need reliable solution you better to design your own PCB or consider other solutions, such as mini-breakout or SparkFun.
