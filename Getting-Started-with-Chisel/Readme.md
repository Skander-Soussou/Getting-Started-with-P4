# Getting started with Chisel
## Chisel Installation
Follow the instructions in the [Chisel template](https://github.com/freechipsproject/chisel-template)
## Create a Chisel program
* Go to chisel-template/src/main/scala
* Create a new folder
* inside it create a new .Scala file
## Generate Verilog
add this code at the end of your program
``` Scala
object HelloWorld extends App {
  chisel3.Driver.execute(args, () => new HelloWorld) // HelloWorld is the name of your module
}
```
In the Terminal write
```
sbt 'runMain nameOfTheFolder.HelloWorld'
```
This command will Generate a Verilog file HelloWorld.V
