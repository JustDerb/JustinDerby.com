JustinDerby.com
===============

Generated by [Hugo](https://gohugo.io/), with the [Blackburn](http://themes.gohugo.io/blackburn/) theme.

Updating
--------

Updating is done via AWS CodeBuild. But, if you need to manually update it:

```bash
hugo
aws s3 cp public/ s3://justinderby.com/ --recursive
```
