# AITPI
Arbitrary Input for Terminal or a Pi, or Aitpi (pronounced 'eight pi')

# Goal
The goal of this project is to provide a simple, but arbitrary, input
mechanism for use with a raspberry pi, or a terminal keyboard (maybe more SBCs in the future?!).

This program can be configured with two simple json files.

# Supported
The project supports:
- Simple 'buttons'
    - '1 to 1' gpio to button setup on a raspberry pi
    - Non interrupt based key input
    - Interrupt based key input (using pynput)
- Encoders
    - '2 to 1' gpio to encoder setup on a raspberry pi
    - Non interrupt based 2 to 1 key input
    - Interrupt based 2 to 1 key input (using pynput)

# Examples
To configure your setup, you can create up to three types of json files:

## Command Registry:
A registry of commands that will interact directly with your user program
```
{
    "type0": {
        "commandName0": {
            "mechanism": "button",
            "id": "0"
        },
        "commandName1": {
            "mechanism": "button",
            "id": "1"
        }
    },
    "type1": {
        "commandName2": {
            "mechanism": "encoder",
            "id": "2"
        }
    }
}
```
- The first layer of json define what 'type' each command is. You can use this to sort your commands in a meaningful way.
    - NOTE: Currently, you need a single type layer and cannot have more. This will be remedied in the future to allow 'foldered' types
- Each command is listed with a name, and a corrosponding dictionary.
    - Each command name must be unique regardless of type (this will be remedied once foldered types are implemented)
    - Each command must have a 'mechanism' and 'id' attribute
        - 'mechanism' lets Aitpi know what type of input this can be connected to
            - Valid mechanisms: 'encoder', 'button'
        - 'id' is the message id that the command events will be sent over
            - Valid ids: Any positive int, negative ints are reserved for Aitpi and could produce bad side effects

## Input list
The list of all 'input units' that your system uses
```
[
    {
        "name": "Button0",
        "type": "button",
        "mechanism": "gpio",
        "trigger": "5",
        "reg_link": "NAME1"
    },
    {
        "name": "Encoder0",
        "type": "encoder",
        "mechanism": "gpio",
        "trigger": "5",
        "reg_link": "NAME1"
    }
]
```
- This is an array of depth 1, with all your 'input units' listed as dictionaries inside
    - "name": specifies the name of the input unit
        - Valid names: Any string, must be unique among all input units
    - "type": specifies what type of input this unit is
        - Valid types: 'button', 'encoder'
    - "mechanism": This tells Aitpi by what mechanism the input will be watched
        - Valid mechanisms: 'key_interrupt', 'key_input', 'gpio'
            - key_interrupt: Uses [pynput](https://pypi.org/project/pynput/) to set interrupts on your keyboard itself
            - key_input: Manual in-code input through the function 'aitpi.takeInput'
            - gpio: Raspberry pi GPIO input, all input units are assumed to be active high
    - "trigger": The key string or gpio number that will trigger input
        - Valid triggers: Any string, or any valid unused gpio number on a raspberry pi
            - Note strings of more than one char will not work with key_interrupt (pynput)
    - "reg_link": This corrosponds to a command from the command registry and will determine what message is sent to your user program

## Foldered Commands
Foldered commands allows you to consider all the files in a folder as a 'command' in the registry.
This uses the [watchdog](https://pythonhosted.org/watchdog/) python package to monitor folders and update on the fly.
All commands added will be deleted and reloaded upon program startup.
```
[
    {
        "path": "/path/to/your/folder",
        "type": "<registry_type>",
        "id": "3",
        "mechanism": "button"
    },
    {
        "path": "/another/path",
        "type": "<registry_type>",
        "id": "4",
        "mechanism": "encoder"
    }
]
```
- This is an array of depth 1 that lists all the folders you want to add
    - "path": Specifies the folder that will be watched
        - Valid paths: Any valid folder on your system
    - "type": This will tell Aitpi where to insert the commands from the folder into your command registry
        - Valid types: Any string
    - "id": When a command is added from the folder, this id will be the command registry 'id' value
        - Valid ids: Any positive int, negative ints are reserved for Aitpi and could produce bad side effects
    - "mechanism": When a command is added from the folder, this mechanism will be the command registry 'mechanism' value


# Example usage:
```python

# import the base aitpi
import aitpi

# The postal service allows us to receive messages
from aitpi.postal_service import PostalService

# In order to receive messages we must have an object with a consume(message) function
# This does not need to be a class.
# This is a simple example of how to implement a 'consumer'
class Watcher():
    def consume(self, message):
        print("Got command: %s" % message.name)
        print("On event: %s" % message.event)
        print("All attributes: %s" % message.attributes)

watcher = Watcher()

# Here we add a consumer that will receive commands with ids 0,1,2,3,4, these ids are the sameconsume
# as defined in your registry json file.consume
PostalService.addConsumer([0,1,2,3,4], PostalService.GLOBAL_SUBSCRIPTION, watcher)

# We must first initialize our command registry before we can start getting input
aitpi.addRegistry("<path_to_json>/command_reg.json", "<path_to_json>/foldered_commands.json")

# We can add multiple registries, and do not need the foldered commands
aitpi.addRegistry("<path_to_json>/another_reg.json")

# Once we initialize our system, all interrupt based commands can be sent imediately.
# Therefore, make sure you are ready to handle any input in your functions before calling this.
aitpi.initInput("<path_to_json>/example_input.json")

# For synchronous input (not interrupt based) using the 'key_input' input mechanism is desireable
# You can setup a custom progromatic form of input using this (If it is good enough, add it to AITPI!)
while (True):
    aitpi.takeInput(input())
```