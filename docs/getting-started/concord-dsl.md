---
layout: wmt/docs
title:  Concord DSL
side-navigation: wmt/docs-navigation.html
---

# {{ page.title }}

The Concord DSL defines the syntax used in the Concord file - a `concord.yml`
file in the root of your project. It is based on using the simple, human
readable format [YAML](http://www.yaml.org/) and defines all your workflow
process flows, configuration, forms and other aspects:

- [Example](#example)
- [Project Configuration in `configuration`](#configuration)
  - [Entry Point](#entry-point)
  - [Dependencies](#dependencies)
  - [Template](#template)
  - [Arguments](#arguments)  
- [Process Definitions in `flows:`](#flows)
  - [Entry points](#entry-points)
  - [Execution steps](#execution-steps)
  - [Expressions](#expressions)
  - [Conditional expressions](#conditional-expressions)
  - [Return command](#return-command)
  - [Groups of steps](#groups-of-steps)
  - [Calling flows](#calling-other-flows)
  - [Error handling](#error-handling)
  - [Setting variables](#setting-variables)
- [Named Profiles in `profiles`](#profiles)

Some features are more complex and you can find details in separate documents:

- [Scripting](./scripting.html)
- [Tasks](./tasks.html)
- [Forms](./forms.html)
- [Docker](./docker.html)

## Example

```yaml
configuration:
  entryPoint: "main"
flows:
  main:
    - log" "Getting started now"
    - task: sendEmail                               # (1)
      in:
        to: me@localhost.local
        subject: Hello, Concord!
      out:
        result: operationResult
      error:
        - log: "email sending error"
    - if: ${result.ok}                              # (2)
      then:
        - reportSuccess                             # (3)
      else:
        - log: "Failed: ${lastError.message}"

  reportSuccess:
    - ${dbBean.updateStatus(result.id, "SUCCESS")}; # (4)
```

In this example:
- the `main` flow is configured as the process starting point with `entryPoint`
- a simple log message is created to start
- the `sendEmail` [task](./tasks.html) (1) is executed using two input
  parameters: `to` and `subject`. The output of the task is stored in `result`
  variable.
- `if` [expression](#expressions) (2) is used to either call `reportSuccess`
  sub-flow (3) or to log a failure message;
- `reportSuccess` flow is calling a Java bean using the EL syntax (4).

The actual task names and their required parameters may differ. Please refer to
the [task documentation](./tasks.html) and the specific task used for details.

<a name="configuration"/>
## Project Configuration in `configuration`

Overall configuration for the project and process executions is contained in the
`configuration:` top level element of the Concord file:

- [Entry Point](#entry-point)
- [Dependencies](#dependencies)
- [Template](#template)
- [Variables](#variables)

### Entry Point

The `entryPoint` configuration sets the name of the flow that will be used for
process executions by default.

```yaml
configuration:
  entryPoint: "main"
flows:
  main:
    - log: "Hello World"
```

### Dependencies

The `dependencies` array allows users to specify the URLs of dependencies such
as 

- Concord plugins and their dependencies 
- dependencies needed for specific scripting language support
- other dependencies required for process execution

```yaml
configuration:
  dependencies:
    - "https://repo1.maven.org/maven2/org/codehaus/groovy/groovy-all/2.4.11/groovy-all-2.4.11.jar"
    - "https://repo1.maven.org/maven2/org/apache/commons/commons-lang3/3.6/commons-lang3-3.6.jar"
```

The artifacts are downloaded and added to the classpath for process execution
and are typically used for [task implementations](./tasks.html).

### Template

A template can be used to allow inheritance of all the configuration of another
project. The value for the `template` field has to be a valid URL pointing to
a JAR-archive of the project to use as template. 

The template is downloaded for [process execution](./processes.html#execution)
and exploded in the workspace. More detailed documentation, including
information about available templates, is available in the
[templates section](../templates/index.html).

### Arguments

Default values for arguments can be defined in the `arguments` section of the
configuration as simple key/value pairs as well as nested values

```
configuration:
  arguments:
    name: "Example"
    coordinates:
      x: 10
      y: 5
      z: 0
flows:
  main:
    log: "Project name: ${name}"
    log: "Coordinats (x,y,z): ${coordinates.x}, ${coordinates.y}, ${coordinates.z}
```

Values of `arguments` can contain [expressions](#expressions). Expressions can
use all regular tasks as well as external dependencies.

```yaml
configuration:
  arguments:
    listOfStuff: ${myServiceTask.retrieveListOfStuff()}
    myStaticVar: 123
```

The variables are evaluated in the order of definition. For example, it is
possible to use a variable value in another variable if the former is defined
earlier than the latter:

```yaml
configuration:
  arguments:
    name: "Concord"
    message: "Hello, ${name}"
```

<a name="flows"/>
## Process Definitions in `flows:`

Process definitions are configured in named sections under the `flows:` 
top-level element in the Concord file.

### Entry Points

Entry point define the name and start of process definitions within the
top-level `flows:` element. Concord uses entry points as a starting step of an
execution. A single Concord file can contain multiple entry points.

```yaml
flows:
  main:
    - ...
    - ...

  anotherEntry:
    - ...
    - ...
```

An entry point must be followed by one or more execution steps.

### Execution Steps

<a name="expressions"/>
#### Expressions

Expressions must be valid
[Java Expresssion Language EL 3.0](https://github.com/javaee/el-spec) syntax 
and can be simple evaluations or perform actions by invoking more complex code. 

Short form:
```yaml
flows:
  main:
    # calling a method
    - ${myBean.someMethod()}

    # calling a method with an argument
    - ${myBean.someMethod(myContextArg)}

    # literal values
    - ${1 + 2}

    # EL 3.0 extensions:
    - ${[1, 2, 3].stream().map(x -> x + 1).toList()}
```

Full form:
```yaml
flows:
  main:
    - expr: ${myBean.someMethod()}
      out: myVar
      error:
        - ${log.error("something bad happened")}
```

Full form can optionally contain additional declarations:
- `out` field: contains the name of a variable, in which a result of
the expression will be stored;
- `error` block: to handle any exceptions thrown by the evaluation.
Exceptions are wrapped in `BpmnError` type.

Literal values, for example arguments or [form](#forms) field values, can
contain expressions:

```yaml
flows:
  main:
    - myTask: ["red", "green", "${colors.blue}"]
    - myTask: { nested: { literals: "${myOtherTask.doSomething()}"} }
```

### Conditional Expressions


```yaml
flows:
  main:
    - if: ${myVar > 0}
      then:                           # (1)
        - log: it's clearly non-zero
      else:                           # (2)
        - log: zero or less

    - ${myBean.acceptValue(myVar)}    # (3)
```

In this example, after `then` (1) or `else` (2) block are completed,
the execution continues with the next step in the flow (3).

To compare a value (or a result of an expression) with multiple
values, use the `switch` block:

```yaml
main:
  - switch: ${myVar}
    red:
      - log: "It's red!"
    green:
      - log: "It's definitely green"
    default:
      - log: "I don't know what it is"
      
  - log: "Moving along..."
```

In this example, branch labels `red` and `green` are the compared
values and `default` is the block which will be executed if no other
value fits.

Expressions can be used as branch values:

```yaml
main:
  - switch: ${myVar}
    ${aKnownValue}:
      - log: "Yes, I recognize this"
    default:
      - log: "Nope"          
```

### Return Command

The `return` command can be used to stop the execution of the current
(sub) flow:

```yaml
flows:
  main:
    - if: ${myVar > 0}
      then:
        - log: moving along
      else:
        - return
```

### Groups of Steps

Several steps can be grouped in one block. This allows `try-catch`-like
semantics:

```yaml
flows:
  main:
    - log: a step before the group
    - ::
      - log: a step inside the group
      - ${myBean.somethingDangerous()}
      error:
        - log: well, that didn't work
```

### Calling Other Flows

Flows, defined in the same YAML document, can be called by their names:

```yaml
flows:
  main:
    - log: hello
    - mySubFlow
    - log: bye

  mySubFlow:
    - log: a message from the sub flow
```

### Error Handling

The full form syntax allows using input variables (call arguments) and supports
error handling:

Task and expression errors are normal Java exceptions, which can be
"caught" and handled using a special syntax.

Expressions, tasks, groups of steps and flow calls can have an
optional `error` block, which will be executed if an exception occurrs.

```yaml
flows:
  main:
  # handling errors in an expression
  - expr: ${myTask.somethingDangerous()}
    error:
    - log: "Gotcha! ${lastError}"

  # handling errors in tasks
  - task: myTask
    error:
    - log: "Fail!"

  # handling errors in groups of steps
  - ::
    - ${myTask.doSomethingSafe()}
    - ${myTask.doSomethingDangerous()}
    error:
    - log: "Here we go again"

  # handling errors in flow calls
  - call: myOtherFlow
    error:
    - log: "That failed too"
```

The `${lastError}` variable contains the last caught
`java.lang.Exception` object.

If an error was caught, the execution will continue from the next step.

```yaml
flows:
  main:
  - expr: ${misc.throwBpmnError('Catch that!')}
    error:
    - log: "A"

  - log: "B"
```

An execution logs `A` and then `B`.

When a process cancelled (killed) by a user, a special flow
`onCancel` is executed:

```yaml
flows:
  main:
  - log: "Doing some work..."
  - ${sleep.ms(60000)}

  onCancel:
  - log: "Pack your bags, boys. Show's cancelled"
```

Similarly, `onFailure` flow is executed if a process crashes:

```yaml
flows:
  main:
  - log: "Brace yourselves, we're going to crash!"
  - ${misc.throwBpmnError('Handle that!')}

  onFailure:
  - log: "Yep, we just did"
```

In both cases, the server starts a "child" process with a copy of
the original process state and uses `onCancel` or `onFailure` as an
entry point.

**Note**: if a process was never suspended (e.g. had no forms or no
forms were submitted), then `onCancel`/`onFailures` receive a
copy of the initial state of a process, which was created when the
original process was started by the server.

This means that no changes in the process state before suspension
will be visible to the "child" processes:

```yaml
flows:
  main:
  # let's change something in the process state...
  - set:
      myVar: "xyz"

  # will print "The main flow got xyz"
  - log: "The main flow got ${myVar}"

  # ...and then crash the process
  - ${misc.throwBpmnError('Boom!')}

  onFailure:
  # will log "I've got abc"
  - log: "And I've got ${myVar}"

configuration:
  entryPoint: main
  arguments:
    # original value
    myVar: "abc"
```


### Setting Variables

The `set` command can be used to set variables in the current flow:

```yaml
flows:
  main:
    - set:
        a: "a-value"
        b: 3
    - log: ${a}
    - log: ${b}
```

<a name="profiles"/>
## Named Profiles in `profiles`

Profiles are named collections of configuration, forms and flows and can be used
to override defaults set in the top-level content of the Concord file. They are
created by inserting a name section in the `profiles` top-level element.

Profile selection is configured when a process is
[executed](./processes.html#execution).

For example, if the process below is executed using the `myProfile` profile, 
the value of `foo` is `bazz` and appears in the log instead of the default
`bar`.

```yaml
flows:
  main:
  - log: "${foo}

configuration
  arguments:
    foo: "bar"
    
profiles:
  myProfile:
    configuration:
      arguments:
        foo: "bazz"
```

The `activeProfiles` parameter is a list of project file's profiles that is
used to start a process. If not set, a `default` profile will be used.

