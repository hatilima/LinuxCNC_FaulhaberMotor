Using hardware PWM, setting the motor directions manually, reading encoder 
values and using PID:

Previously...
On hardware pwm:
hm2_5i23.0.pwmgen.00. 	- pwm on phy pin 007

For direction:
set gpio.012 as output
set gpio.013 as output
create net of hm2_5i23.0.gpio.012.out mapped to dir1
create net of hm2_5i23.0.gpio.013.out mapped to dir2

Then the controls to the PWM are:
-hm2_5i23.0.pwmgen.enable
-hm2_5i23.0.pwmgen.value
-hm2_5i23.0.gpio.012.out (1 or 0)
-hm2_5i23.0.gpio.013.out (1 or 0)


Then setting up encoders:
-one encoder is defined in hardware (hm2_5i23.0.encoder.00.). Of course
there is an option for software encoders in HAL but they have a lower
maximum count rate...range of 10KHz to 50KHz while hardware rates can go
to MHz. The software encoder is simply loadrt encoder num_chan=num
-the first HostMot2 encoder (encoder.00) by default goes to physical 
pins 002 for chanB and 003 for chanA
-for the encoder to be read, the HostMot2 function hm2_5i23.0.read has to
be added to the servo-thread.

Then setting up PID
This requires loading pid hal module, adding its functions to a thread,
connecting pid pins to other component pins and signals, setting PID
gains (used P=1, I=1 and D=0). Changing the gains show oscillations that
are sharp for too high P (above 7) and slow for too low P (below 0.5).

We forego the position PID that was implemented above and attempt a velocity
control PID. This is done by getting velocity from encoder, i.e
hm2_5i23.0.encoder.00.velocity, and feeding it as pidx.feedback and then
getting pidx.output to drive hm2_5i23.0.pwmgen.00.value. The reference 
velocity is applied through a pin of the PID module called pidx.command.
For my test, setp pidx.Pgain 0.07 and then setp pidx.Igain 0 and
setp pidx.Dgain 0. With these settings, I was changing between velocities
by sets x-velcmd 40 or -40 or 0 etc; where x-velcmd is mapped to pin
pidx.command using
net x-velcmd       		=>  pidx.command

The above method for commanding the velocity is replaced with siggen so
as to see the step response of the system using a squarewave.
The frequency of the squarewave is set to 0.5Hz and the amplitud at 400.
This means that the reference input will be a "velocity target" of 400 units per second.
Encoder scale is set to 100, as increasing it makes the PWM to saturate before
the velocity even reaches the command signal of 400 units per second. 
With Encoder Scale at 200, pwmgen is spitting out 100% duty cycle and motor running
max speed with a P gain of just 0.3, but the velocity that is being spit out by
the encoder is not even 70% of reference input.
The Pgain is set to around 0.225....0.23 is also ok from trials.
