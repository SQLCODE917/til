# Create A New File In A New Directory

Form within a vim session, create a buffer for a new file in a directory that doesn't exist:

```
:e vim/create-a-new-file-in-a-new-directory.md
```

The containing directory doesn't exist, so let's create it with a combination of ViM filename shorthands and shelling out to the `mkdir` command:

```
:!mkdir -p %:h
```

The `%` is a shorthand for the qualified path of the current file.
The `:h` is a filename modifier that returns the *head of the filename*, that is, it resolves to the path with everything except the name of the file.
Thus, this command is essentially resolving to:

```
:!mkdir -p vim/
```

ViM will shell out with this command making directories for all non-existent directories in the given path.
Now you can happily save your new file
