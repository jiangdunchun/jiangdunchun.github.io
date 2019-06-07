# A Simple MR Application about Digital Twins and It's Great Expectation
*7/6/2019*

Based on Hololens and Unity3D, we developed a simple MR application could monitor and control a PLC system: showing the real-time data (tempreture, humidity, CO2 concentrations) from the sensors and controlling the devices (lamps, fans, pumps) in virtual world. In academia, this technology is called Digital Twins. Here is our demo:

<center><video src="blogs/A_Simple_MR_Application_about_Digital_Twins_and_It's_Great_Expectation/demo.mp4" width="80%" controls="controls"></video></center>

But this application's design is not as simple as we guessed. The below graph explains how this application works. We insert a Real-time Data Manager as intermediation to avoid setting up a directive connection between the Hololens and PLC. This intermediation acquires the data from PLC based on ModbusTCP, lables these data with unique tags, and opens a websocket server allowed the clients to get these data by unique tags.

<center><img style="max-width: 60%;" src="blogs/A_Simple_MR_Application_about_Digital_Twins_and_It's_Great_Expectation/holo.png"></center>

Even though this may seem redundant, I have my own thought. After several similar programs about Digital Twins, we have already accumulated lots of valuable experience, and summarized out the common framework from these programs in the graph below. The DataPool Service has been supported to acquiring real-time data from several types of data sources by standard protocols (OPC, ModbusTCP) or customized communication ways (we designed a subprotocol based on websocket for monitoring and controlling the GPIO state in Respberry Pi). For this reason, the display terminals could access to different types of data sources just by supporting the websocket which is much easier than realising industrial communication protocols. As a deamon, the DataPool Service also contains a database storing these real-time data with timestamps.

<center><img style="max-width: 60%;" src="blogs/A_Simple_MR_Application_about_Digital_Twins_and_It's_Great_Expectation/data_pool.png"></center>

But it is still far from my destination. This framework still haven't applied to a big program contains tens of thousands of data sources, whose characteristics of high-density throughput and computation might need to be resolved by using distributed system. And it also hasn't made maximum use of the history data. I plan to add an artificial intelligence module to this system for real-time diagnosis and prediction based on these history data.

