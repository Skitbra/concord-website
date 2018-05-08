---
layout: wmt/docs
title:  Resource Task
side-navigation: wmt/docs-navigation.html
---

# {{ page.title }}

The `resource` task provides methods to read and write the resource in `json` or `string` format.

The task is provided automatically by the Concord and does not
require any external dependencies.

<a name="usage"/>

## Usage

### Reading a resource
`asJson` method is used to read the resource as `json` object.
```yaml
- flows:
    default:
    - expr: ${resource.asJson('sample-file.json')}
      out: jsonObj
    # we can now use it like a simple object
    - log: ${jsonObj.any_key}
```
`asString` method is used to read the resource as `string` object.
```yaml
- log: ${resource.asString(sample-file.txt)}
```

### Writing a Resource
`writeAsJson` method is used to write the resource as `json`.
```yaml
- flows:
    default:
    - set:
       newObj:
        name: testName
        type: testType
    # write and print the relative path of a file
    - log: ${resource.writeAsJson(newObj)} 
```
`writeAsString` method is used to write the resource as `string`.
```yaml
# write and print the relative path of a file
- log: ${resource.writeAsString('test string')} 
```

`writeAsJson` and `writeAsString` methods return a path to the temporary file.