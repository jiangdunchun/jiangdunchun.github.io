# WampFramework: Remotely Invoke the API in Local C# Assembly
*9/5/2019*

Last November, I got a task about refactoring a Remote Rendering Framework. It was a short-term project and contructed by large amount of temporary code. The graph below presents the framework of this project. The most chaotic part is the Communication Processing Layer, which receives the message(for example the responding to events of mouse and keyboard) from HTML, processes them and sends the return message back(for example the result image of rendering). In this module, We used to invoke the target interfaces of Business Logic Layer by analysing the message manually, which means we need to add an option in the switch function of message analysis everytime when a new interface was opened up, and some other essential code. 

![Remote_Rendering_Framework](blogs/wamppic/Remote_Rendering_Framework.png)

The simplification of this matter is how to design a flexible module of RPC(Remote Procedure Call) and Pub&Sub(Publisher and Subscriber) models. I found [WAMP](https://wamp-proto.org), a intriguing solution to this problem. The graph below shows the WAMP in an IoT application.

![WAMP_in_an_IoT_application](blogs/wamppic/WAMP_in_an_IoT_application.svg)

 The great idea of WAMP really inspired me, but I still need to do some changes. Compared with the application in IoT, the Callee and Publisher in our program are not running on the remote computer. We must use the memory to tranfer the data between Router and Callee(Publisher) rather than network packet, because the result images of rendering were always larger than 1MB per second(I will introduce my effort of compressing these data in another blog). So, I set up the goals of this task:

>* Open the API of c# assemblies to websocket connection in a simple way, while the developers don't need to concentrate on the network packet anymore

>* Export all c# API to js file, as a result, the front_end could invoke these API just by referencing these files

