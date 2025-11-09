Optimised for v463 (early 2021).

```sh
sed -e '/-- SUBSTITUTETHIS$/ {' -e 'r lib.lua' -e 'd' -e '}' template.rbxmx > LIB.RBXMX
```

Add your resultant [`LIB.RBXMX`](LIB.RBXMX) to your `AssetCache` and change the file name to some numerical iden. For example, you can rename it to `00000001234`.

Then, at the beginning of each script in your game, call `require(1234)`.
