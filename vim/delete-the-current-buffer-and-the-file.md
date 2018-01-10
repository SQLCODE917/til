# Delete The Current Buffer And The File

ViM has it's own `[delete()](http://vimdoc.sourceforge.net/htmldoc/eval.html#delete%28%29)` function for deleting files.
It's nice because it's portable.

If you want to just delete the current file, you can use the `[%](http://vimdoc.sourceforge.net/htmldoc/change.html#quote%25)`, current file, register:

```
:call delete(expand('%'))
```

or

```
:call delete(@%)
```

To delete the current buffer as well, append the `[bdelete](http://vimdoc.sourceforge.net/htmldoc/windows.html#:bdelete)`:

```
:call delete(expand('%')) | bdelete!
```

And if you do this kind of thing a lot, you can map it to something like the familiar `rm`:
In your `.vimrc`

```
nnoremap rm :call delete(expand('%')) \| bdelete!<CR>
```
