# Setting up a postgresql database on an ubuntu server 
`!!!!Throughout this demostration, i have used a linux user with the username of bee`
```
sudo apt update
```

```
sudo apt install libpq-dev postgresql postgresql-contrib nginx curl
```

Activate the virtual environment 
```bash
source /home/~username~/venv/bin/activate
```

install psycopg2-binary
```
pip install psycopg2-binary
```

### Login to a psql session 
```
sudo -u postgres psql
```

### Create a databawe 
```
CREATE DATABASE nnra_circle;
```

## Create a database user for the project
```
CREATE USER bee WITH PASSWORD 'bee';
```
Where bee serves as the username 


```
ALTER ROLE bee SET client_encoding TO 'utf8';
ALTER ROLE bee SET default_transaction_isolation TO 'read committed';
ALTER ROLE bee SET timezone TO 'UTC';
```

```
GRANT ALL PRIVILEGES ON DATABASE nnra_circle TO bee;
```

Exit psql session using 
```
\q
```


#### Open settings.py located at `/home/bee/NNRA-circle/nnra_circle/nnra_circle/settings.py`

Locate the database configuration section and in the `else` clause, paste in your database configuration as follows 

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'nnra_circle',
        'USER': 'bee',
        'PASSWORD': 'bee',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```


