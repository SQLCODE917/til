# Share Common Libraries Between Components

Serverless provides a facility for sharing code between functions in a component - the component-level _lib_ folder.
You can read about it [in the docs](http://docs.serverless.com/docs/project-structure).

It does not provide a facility for sharing code between components.
Perhaps it's implying that your components should be so diverse that there should not be any common functions between them.
Perhaps.
For our enterprise application, we definitely have shared logic between components, 
and we are going to use a feature of the [serverless-optimizer-plugin](https://github.com/serverless/serverless-optimizer-plugin).

First, create a folder for the shared libraries at the root of your project and put one there:

```
s-project.json
s-resources-cf.json
...
lib
    |_projectLevelSharedLib.js
component1
    ...
    |_function1
        ...
        |_s-function.json
        ...
```

Second, for each function that is going to make use of the shared library, 
enable the _serverless-optimizer-plugin_ in it's _s-function.json_:

```
"custom": {
    "optimize": {
        "minify": true,
        "requires": [
            {
                "name": "../../lib/projectLevelSharedLib",
                "opts": {
                    "expose": "./lib/projectLevelSharedLib"
                }
            }
        ]
    }
}
```

Third, for each function that is going to make use of the shared library,
include the exposed path at the top of it's handler:

```
const projectLevelSharedLib = require('./lib/projectLevelSharedLib');
```

Good to go!
