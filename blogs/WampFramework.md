# WampFramework: Remotely Invoke the API in C# Assembly
*9/5/2019*

Last November, I got a task about refactoring the code of a Remote Render Framework. It was a short-term project and contructed by large amount of temporary code. The graph below presents the framework of this project. The most chaotic module is the Communication Processing Layer, which receives the message from HTML, processes them and sends the return message back. In this module, We used to switch and invoke the target interfaces of Business Logic Layer by analysing the message manually, which means we need to add an option in the switch function of message analysis everytime when a new interface was opened up, and some other esential code. 

![avatar](blogs/wamppic/1.png)

