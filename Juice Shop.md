# **DOM XSS on search field**

### 1. Go to the root page of the application
### 2. In the search field paste the following payload

```html
<img src=x onerror=alert(1)>
```

<img width="1609" height="767" alt="Screenshot_20260413_203236" src="https://github.com/user-attachments/assets/6a25ba39-67c8-4ff3-8c95-89d4fd5e56a8" />


## SQLi on login form, allowing authenthication bypass

### 1.Create an account and then go to the login page

### 2. On the email field enter your valid email and append `' --` payload at the end.

### 3.On the password field enter some invalid password, that don't match your actual password.

### 4.Click login, and you'll se that you bypassed authenthication through SQLi

### **Note:**
> Same vulnerbility can bring to privilege escaltion, if you use ' OR 1=1-- in the empty email field, gaining access to admin fucntionality

<img width="1003" height="654" alt="Screenshot_20260413_203946" src="https://github.com/user-attachments/assets/8793038f-fb29-438a-9635-9d8eee7457bb" />


## JWT none algorithm attack

### 1. Create two accounts and login at one of them

### 2. Go to /profile endpoint(go to your account profile page)

### 3. Notice that your request contains a JWT token, base64 decode it.

<img width="760" height="756" alt="Screenshot_20260413_205307" src="https://github.com/user-attachments/assets/565129a4-6f23-49cd-aafc-bb8a079c4b51" />


### 4. Note that the id field seems very interesting, now logout from this account and then login to the second account, do the same operations and remember the id of this account(in my case it's 23)

### 5.Login to the your first accont and change the id field, to the id of the second account

### 6.Change the alg field to none and delete the signature header.

### 7. Issue the request with this new JWT and you can see than you got access to your second account




## BOLA on basket functionality

### 1. Navigate to your basket and cathc this request with burp(/rest/busket/{id})

### 2.Change the id parameter of your busket.

### 3. Notice that you can view another user busket, without any authorization

#### My basket before an attack

<img width="1520" height="711" alt="Screenshot_20260414_111951" src="https://github.com/user-attachments/assets/29e9e554-62b1-4f87-8ca1-ca97abf44b23" />


#### And after an attack

<img width="1536" height="748" alt="Screenshot_20260414_112027" src="https://github.com/user-attachments/assets/8da52564-fffb-4884-befe-633c2470bd52" />


## BOLA at order confirmation

### 1. Configure your account by adding, a product on basket, adding card

### 2. Then place an order to this product and catch this request with Burp(/rest/busket/{id}/checkout)

### 3. Change the {id} in request and you can see, that you have an access to another user order without any authorization.

#### Order before an attack

<img width="1537" height="754" alt="Screenshot_20260414_113645" src="https://github.com/user-attachments/assets/1bd09059-1bcb-4eeb-a47a-be8c3b4a7c22" />


#### And after an attack


<img width="1536" height="765" alt="Screenshot_20260414_113705" src="https://github.com/user-attachments/assets/853e973d-5e4a-4d76-9a1e-898364711e93" />



## FTP direcotry exposure

### Navigate to http://localhost:3000/ftp

