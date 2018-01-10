# Change Tab Spacing In TextEdit

What if you want to `set ts=4`, only in the default OSX text editor?
Like a real hacker, open your Terminal and throw down this ditty:

```
defaults write com.apple.TextEdit "TabWidth" '4'
```

Where `'4'` is the number of spaces a tab should be.
Restart your TextEdit.

Baller
