# Recovery

## Recovering admin account

Generate a bcrypt hash.

replace `supersecret` with your desired password.

```
docker run -it --rm python:3 /bin/bash
pip install bcrypt
python -c "import bcrypt;print(bcrypt.hashpw(b'supersecret', bcrypt.gensalt()))"
```

One line:

```
docker run python:3 /bin/sh -c "pip install bcrypt && python -c \"import bcrypt;print(bcrypt.hashpw(b'supersecret', bcrypt.gensalt()))\""
```

Now push the bcrypt hash into the Postgres database of Cog:

```
UPDATE users SET password_digest='<calculated digest>' WHERE username = 'admin';
```
