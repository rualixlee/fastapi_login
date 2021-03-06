# FastAPI-Login

FastAPI-Login tries to provide similar functionality as [Flask-Login](https://github.com/maxcountryman/flask-login) does.

## Installation

```shell script
$ pip install fastapi-login
```

## Usage

To begin we have to setup our FastAPI app:
````python
from fastapi import FastAPI

SECRET = "your-secret-key"

app = FastAPI()
````
To obtain a suitable secret key you can run ``import os; print(os.urandom(24).hex())``.

Now we can import and setup the LoginManager, which will handle the process of
encoding and decoding our Json Web Tokens.

````python
from fastapi_login import LoginManager
manager = LoginManager(SECRET, tokenUrl='/auth/token')
````
For the example we will use a dictionary to represent our user database. In your
application this could also be a real database like sqlite or Postgres. It does not
matter as you have to provide the function which retrieves the user.

````python
fake_db = {'johndoe@e.mail': {'password': 'hunter2'}}
````

Now we have to provide the ``LoginManager`` with a way to load our user. The 
`user_loader` callback should either return your user object or ``None``

````python
@manager.user_loader
def load_user(email: str):  # could also be an asynchronous function
    user = fake_db.get(email)
    return user
````

Now we have to define a way to let the user login in our app. Therefore we will create
a new route:

````python
from fastapi import Depends
from fastapi.security import OAuth2PasswordRequestForm
from fastapi_login.exceptions import InvalidCredentialsException

@app.post('/auth/token')
def login(data: OAuth2PasswordRequestForm = Depends()):
    email = data.username
    password = data.password

    user = load_user(email)  # we are using the same function to retrieve the user
    if not user:
        raise InvalidCredentialsException  # you can also use your own HTTPException
    elif password != user['password']:
        raise InvalidCredentialsException
    
    access_token = manager.create_access_token(
        data=dict(sub=email)
    )
    return {'access_token': access_token, 'token_type': 'bearer'}
````

Now whenever you want your user to be logged in to use a route, you can simply
use your ``LoginManager`` instance as a dependency.

````python
@app.get('/protected')
def protected_route(user=Depends(manager)):
    ...
````

If you also want to handle a not authenticated error, you can add your own subclass of Exception to the LoginManager.
````python
from starlette.responses import RedirectResponse

class NotAuthenticatedException(Exception):
    pass

# these two argument are mandatory
def exc_handler(request, exc):
    return RedirectResponse(url='/login')

manager.not_authenticated_exception = NotAuthenticatedException
# You also have to add an exception handler to the your app instance
app.add_exception_handler(NotAuthenticatedException, exc_handler)
````
