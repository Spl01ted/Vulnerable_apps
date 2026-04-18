# DVAPI

## API1:2023 Broken Object level authorization

### 1.Create DVAPI account and the login.

### 2. Go to your profile page and notice the request with your username parameter

```http
GET /api/getNote?username=user123 HTTP/1.1
Host: 192.168.1.9:3000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://192.168.1.9:3000/profile
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiI2OWRmM2IyNTQ4OTRiY2IxOGQ5MTNiNDUiLCJ1c2VybmFtZSI6InVzZXIxMjMiLCJpc0FkbWluIjoiZmFsc2UiLCJpYXQiOjE3NzYyMzc0MDF9.4sf3j4zLNfVeDAn0Htxsa0l9MeIQb7C-492HG11fGu0
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Cookie: auth=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiI2OWRmM2IyNTQ4OTRiY2IxOGQ5MTNiNDUiLCJ1c2VybmFtZSI6InVzZXIxMjMiLCJpc0FkbWluIjoiZmFsc2UiLCJpYXQiOjE3NzYyMzc0MDF9.4sf3j4zLNfVeDAn0Htxsa0l9MeIQb7C-492HG11fGu0
If-None-Match: W/"1e-1Ki3Lutz8Ipr70kRGHIS19DiWy4"
Priority: u=4
```

### 3.Go to the Scoreborad page and notice the admin user

### 4.Change your username parameter to admin


## API2:2023 Broken Authentication

### 1.Create DVAPI account and go to your profile at /api/profile endpoint.

>  We will use [This repository](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list) to try brute force JWT secret with hashcat

### 2. Run the following command to brute force

```bash
hashcat -a 0 -m 16500 <Your-JWT-here> /path/to/downloaded/list
```

If you did everything correct, then the secret should be _secret123_

### 3.Then base64 encode this secret and go to the JWT editor extension in Burp.

+ Click _new symmetric key _
+ Click Generate
+ Replace the value of the k property with base64 encoded secret

### 4.Then go to the repeater tab with your request, then to _Json Web token_ tab 

+ Change the value of _isAdmin_ to true
+ Then click _Sign_ at the bottom
+ Sign with your generated key

### 5.Replace your old JWT with this new JWT and click send


## API3:2023 Broken Object property level authorization

## 1.Register an account and by registration add an additional JSON field

```json
"score":2000
```

## 2.Login to that account

## 3. Go to the scoreboard page and you will see that the score of your user is 2000


## API4:2023 Unrestricted resource consumption

## 1. Login to the application

## 2. Go to the profile page and upload a file for example with 20Mb size image.


## API5:2023 Broken Function Level Authorization

## 1.Login to your account
## 2.Go to the scoreboard page and click on of the users and catch the request _GET /api/user/{user}_

## 3. Change the request method to OPTIONS to see what methods are allowed

#### Notice that the DELETE method is allowed

## 4. Make a request with DELETE method and you can see that you can delete this user.


## API6:2023 Server side request forgery

## 1.Login to your account

## 2.Go to your profile page

## 3. In add note functionality add the following URL

`http://localhost:8443`


## API7:2023 Security Misconfiguration

## 1.Login to your account

## 2.Send request staying logged in by sending with invalid JWT token(e.g changing the "isAdmin" field to true).


## API8:2023 Unsafe consumption of APIs

## 1. Go to the login page and catch the request

## 2. Set username to admin and in the password field set following NoSQLi payload

```json
{"username":"admin","password":{"$ne":null } }
```


## API9:2023 Lack of protection from automated threats

## 1. Login to your account

## 2. In the adding ticket functionality add some ticket and capture this request.

## 3. Send this request to Turbo intruder and start attack

## 4. You can see that there is no rate limitings


## API10:2023 Improper assets management

## 1.Login to your account

## 2. Go to the challanges page with `released:1` JSON

## 3. Change the `released` to `unreleased`

