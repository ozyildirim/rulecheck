# Rule Check
Rule Check (aka rulecheck or source rule check) is a command line system for running custom static analysis rules on C, C++, and Java code. The original intended use case is for checking a code base against coding style or coding standard rules. 

Rule Check uses [srcml](https://www.srcml.org/) to parse the source code into XML and then invokes each rule's methods as appropriate to allow the rules to inspect any source code element of interest to the rule. This architecture minimizes duplicate parsing time and allows rule authors to focus on their rule logic instead of the logic needed to parse source code.

Features include:
* Support for parsing C, C++, Java source
* Supports parsing C and C++ prior to preprocessor execution (parses code in the form the developer uses)
* Supports custom rules
  * Groups of rules can be created and published in 'rulepacks'
  * Projects can have custom rules within their own code base (no need to publish/install rules)
  * Rules can have their own custom settings. Rule check will provide the settings to the rule via its standard config file format.
* Supports multiple config file inputs
  * Projects can use an hierarchical set of configurations allowing organizations to provide rules across projects
* Standardized output format for all rules
* Supports ignore list input to ignore specific rule violations in a code base without modifying the code
* Source to be analyzed specified in glob format

Features to be developed include:
* Ability to ignore rule violations by using comments in the source code to turn off/on rules.


___

### Contents
___
* [Installation](#installation)
* [Running](#running)
* [Config](#config)
* [Creating Rules](#creating-rules)
* [Resources](#resources)

___
### Installation

Ensure Python 3.8 or greater is present on the system (see below) and then run:
```
git clone https://github.com/e-shreve/rulecheck
cd rulecheck
pip install .
```

#### Dependencies

##### Python
Python 3.8 or greater is required.

##### lxml
The python xml library lxml is used over the built-in ElementTree library due to speed and additional functionality such as the ability
to obtain the line number of tag from the source XML file. lxml has been available in Wheel install format since 2016
and thus should not present an issue for users. lxml will be installed by pip automatically when insalling rulecheck.

##### srcml
srcml is a source code to xml parser that can parse C, C++, C Preprocessor, C#, and Java code. The pip install of rulecheck will not
install srcml. Find installation files at https://www.srcml.org/ and install on your system.
Version required: 1.0.0 or greater.
For easiest use, srcml should be on the path. Otherwise, the path to srcml can be provided when starting rulecheck from the command line.

___
### Running
___

```
rulecheck --help
```

Note that extensions are case sensitive and .C and .H are by default treated as C++ source files whereas .c and .h
are treated as C source files.

___
### Config
___

#### Selecting Rules

Rule selection is done by specifying one or more rule configuration files on the command line, using the -c or --config option. To specify more than one configuration file, use the config option on the command line once for each configuration file to be read.

Note that rule selection is additive across all configuration files specified. Thus, if config1.json activates ruleA and config2.json activates RuleB then both RuleA and RuleB will be active.

Example of specifying more than one configuration file:

```bash
rulecheck -c config1.json -c config2.json ./src/**/*
```

Rule configuration files are json files, with the following structure:

```JSON
{
  "rules": [
    {
       "name" : "rulepack1.ruleA",
       "settings" : {
          "opt1" : "dog"
       }
    },
    {
       "name" : "rulepack1.ruleB",
       "settings" : {
          "opt1" : "cat"
       }
    }
  ]
}
```

At the top level, an array named "rules" must be provided. Each member of this array is a rule object. 

Each rule object must consist of a "name" string. The name may contain '.' characters which are used to differentiate between 
collections of rules known as rulepacks.

Optionally, a rule object may include a settings object. The expected/supported content of the settings object will depend on the rule. 

Note that rules *may* support being specified multiple times. For example, a rule for finding banned terms or words could support multiple instantiations each with a different word or term specified:

```JSON
{
  "rules": [
    {
       "name" : "rulepack1.bannedword",
       "settings" : {
          "word" : "master"
       }
    },
    {
       "name" : "rulepack1.bannedword",
       "settings" : {
          "word" : "slave"
       }
    }
  ]
}
```

Some rules *may*, however, throw an error if configured more than once. Consult the documentation for a rule for usage instructions. 

To prevent running the same rule multiple times, rulecheck will not load a rule twice if it has the *exact* same settings. In the following run, rulecheck will only load the bannedword rule twice, despite it being specified three times.

```bash
rulecheck -c config1.json -c config2.json ./src/**/*
```

Where config 1.json contains:

```JSON
{
  "rules": [
    {
       "name" : "rulepack1.bannedword",
       "settings" : {
          "word" : "slave"
       }
    }
  ]
}
```

And config2.json contains:

```JSON
{
  "rules": [
    {
       "name" : "rulepack1.bannedword",
       "settings" : {
          "word" : "master"
       }
    },
    {
       "name" : "rulepack1.bannedword",
       "settings" : {
          "word" : "slave"
       }
    }
  ]
}
```

Rulecheck's ability to load multiple configuration files and combine them supports a hierarchical configuration structure. For example, a company may provide a rule set and standard config at the organization level. A team may then apply additional rules and config file for all of their projects. Finally each project may have its own config file. Instead of needing to combine all three config files into a single set (and thus force updates to each project when a higher level policy is changed), rule check can be given all three config files and it takes care of combining the configurations.


#### Specifying Files to Process

The files to process and/or the paths rulecheck will search to find files to process are provided on the command line as the last parameter (it must be the last parameter.) 
The paths are specified in glob format. Recursive searches using '**' are supported. 
In addition, the '?' (any character), '*' (any number of characters), and '[]' (character in range) wildcards are supported.

Multiple files and paths to search can be specified by separating them with spaces. If a space is in a path, enclose the glob in quotation marks.

Alternatively, the files or paths to check can be specified via stdin. Specify '-' as the final parameter to have rulecheck read the list in from stdin.

When searching the paths specified, rulecheck will process any file found with one of the following case-sensitive extensions:
.c, .h, .i, .cpp, .CPP, .cp, .hpp, .cxx, .hxx, .cc, .hh, .c++, .h++, .C, .H, .tcc, .ii, .java, .aj, .cs

To change the list of extensions rulecheck will parse when searching paths, use the -x or --extensions command line option.

#### Specifying Where Rule Scripts Are

Rules are encouraged to be installed onto the python path using a concept known as "rulepacks." This is covered later in this document. 
However, there are situations where rules may not be installed to the python path. For example, when a rule is under development or when a rule is
created for a single project and is kept in the same revision control system as the source being checked by the rule. For these situations, one or more
paths to rules may be specified on the command line using the -r option. If more than one path is needed, repeat the option on the command line for
each path.

Note that the name of a rule specified in a configuration file may contain part of the path to the rule script itself. For example, if

```JSON
	"name" : "rulepack1.ruleA"
```

is in a configuration file, rulecheck will look for a 'rulepack1/ruleA.py' script to load on the path. 

#### Using Ignore List Files
To be written.

#### Options For Controlling srcml

* '--srcml' to specify the path to the srcml binary. Use this option if srcml is not on the path.
* '--tabs' specifies number of spaces to use when substituting tabs for spaces. This impacts the column numbers reported in rule messages.
* '--register-ext' specifies language to extension mappings used by srcml.
* '--srcml-args' allows for specification of additional options to srcml. Do not specify --tabs or -register-ext options here as they have their own dedicated options described above. This option must be provided within double quotation marks and must start with a leading space.


#### Other Options

* '--Werror' will promote all reported rule warnings to errors.
* '--tabs' specifies number of spaces to use when substituting tabs for spaces. This impacts the column numbers reported in rule messages.
* '-v' for verbose output.
* '--version' prints the version of rulecheck and then exits.
* '--help' prints a short help message and then exits.

___
### Rulepacks
To be written...
This section will describe the concept of rulepacks and provide a bit of the technical context for how they work (python path).

___
### Creating Rules

Naming/Path
Extending Rule
What to do in __init__, how settings work
getRuleType

Note: parsing xml, the visit methods must be named visit_xml_nodename_start|end. The use of xml_ at the start
avoids collisions with visit_file_open and visit_file_line as XML does no allow any nodename to start with 'xml'.

- [ ] Document in readme that rules can be loaded multiple times. And that to prevent that a rule should throw an error on init if 2nd time called.

___
##### SrcML Tags
___
##### Tips
Use global tabs setting from command line instead of per-rule setting for tabs
How to prevent rule from being instantiated twice

___
### Resources
___
* [srcml](https://www.srcml.org)
* [srcml source](https://github.com/srcML/srcML)
* [lxml](https://lxml.de/)