# owlsat-hardware
Hardware repo - based on OakwoodEngineering's Obi Wan Komputer

# Tools needed
KiCAD - FOSS EDA software from https://www.kicad.org/download/ 

LTSPICE - Circuit simulator for Windows and MacOS, works well under Wine with Linux (https://www.analog.com/en/resources/design-tools-and-calculators/ltspice-simulator.html)

(or alternatively use Falstad (https://www.falstad.com/circuit/) as it's a simpler interface and just as good for simple circuits that don't require specialized ICs)

git - FOSS version control software. Use git bash for Windows and optionally github desktop of another GUI for easier management.

# To-do (this is somewhat still a jumble of useful ideas)

- ~~Copy OWK repo hardware~~
- figure out what parts to keep and what to shave off
- consolidate everything we need into one board
- use ADM1176 properly or remove (per mlyle's suggestion)
- make transimpedance amp EUV measurement subsystem
- add magnetorquer subsystem (max22200 full bridge driver)
- improve solar charging subsystem if needed (test existing boards first!)
- make MRAM work with RP2350 (important, but requires bootloader and possibly sdk modification (and I don't yet know how to do this))
- change connectors to be less expensive but still reliable (however we will likely still have to use something expensive for the important connectors like battery)
- reorganize power domains
- re-layout board
