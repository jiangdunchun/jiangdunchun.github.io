# A Simple MR Application about Digital Twins and It's Great Expectation
*10/6/2019*

Based on Hololens and Unity3D, we developed a simple MR application could monitor and control a PLC system: showing the real-time data (tempreture, humidity, CO2 concentrations) from the sensors and controlling the devices (lamps, fans, pumps) in virtual space. Here is our demo:

<center><video src="blogs/A_Simple_MR_Application_about_Digital_Twins_and_It's_Great_Expectation/demo.mp4" width="80%" controls="controls"></video></center>

But this application's design is not as simple as we guessed. The below graph explains how this application works. We inserted a Real-time Data Manager as intermediation to avoid setting up a directive connection between the Hololens and PLC. The advantages of this design is obvious: 

>* 

<center><img style="max-width: 60%;" src="blogs/A_Simple_MR_Application_about_Digital_Twins_and_It's_Great_Expectation/holo.png"></center>