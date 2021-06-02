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

This sets up a PID control for position...without velocity. 
This is done by getting position from encoder, i.e
hm2_5i23.0.encoder.00.position, and feeding it as pidx.feedback and then
getting pidx.output to drive hm2_5i23.0.pwmgen.00.value. The reference 
position is applied through a pin of the PID module called pidx.command.
..with poscmd mapped to pin pidx.command using
net poscmd       		=>  pidx.command

Interesting observations:
- Encoder Scale is ticks per unit travelled. It scales the raw ticks from 
the encoder such that the position output from the encoder corresponds to
the units travelled by the encoder. For example, since the encoder has 38912 ppr
and 155648 counts per rev in quadrature mode, and I want to use the outer
circumference of the gear head which has a circumference of 94.25mm.
**In order to give command, poscmd, in ticks, the encoder scale is set to 1. 
This means that an input of 155648 will make one rev of the encoder (motor shaft).
Conversely, an encoder scale setting of 2 will intepret a command of 155648
as two revs. Note that the P gain must be reduced to say 0.001.
**In order to give command in linear mm travelled by the wheel, the encoder 
scale is set to 1651 and the P gain to about 10. This means that the ticks 
will be divided by 1651, resulting into 94.27 reference input per revolution.
This is equivalent to circumference of gear wheel travelling 94.27 and
doing one revolution. P gain of 50 works ok but with velocity overshoots,
P gain of 1 works ok but speed slows at the end AND the motor cannot easily
hold position.

So, for scale = 1, Kp = 10. For scale = 1651, Kp = 10. 
