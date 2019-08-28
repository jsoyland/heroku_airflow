# Run Airflow on Heroku
This is some documentation on how to quickly get Apache Airflow up and running on Heroku.

1. Install or setup supported python version (I'm using [pyenv](https://github.com/pyenv/pyenv) so I just set the desired version in the project directory):
echo "3.6.4" > .python-version

1. Create Python virtual environment to install Airflow along with dependencies
python3 -m venv .venv
source .venv/bin/activate

1. Install airflow, install cryptography module, and set Procfile to init db on initial run
pip install "apache-airflow[postgres, password]"
pip install "cryptography"
pip freeze > requirements.txt

1. Create a `.gitignore` file
    ```
    echo .venv/ > .gitignore
    ```

1. Initialize the git repository and create the Heroku app with a postgres add-on:
    ```
    git init
    git add .
    git commit -m "initial commit"

    heroku create
    heroku addons:create heroku-postgresql:hobby-dev
    ```

1. Setup airflow.cfg
  We will use `airflow.cfg` for most of our application configuration, but any secure values should be kept as Heroku config variables.  The `airflow.cfg` in this repository is already making use of the `DATABASE_URL` that was assigned when we created the database, but we will need a fernet key.  You can generate/set one thusly:
    ```
    heroku config:set AIRFLOW__CORE__FERNET_KEY=`dd if=/dev/urandom bs=32 count=1 2>/dev/null | openssl base64`
    ```
You'll also need to set `AIRFLOW_HOME` to `/app` so that Airflow knows where the `airflow.cfg` file is.  Otherwise when the database initializes it will do so using sqlite, which on Heroku will only be created on an ephemeral file system that has the lifetime of the dyno running it:
    ```
    heroku config:set AIRFLOW_HOME=/app
    ```

1. Set Procfile to initdb on initial run
Heroku uses a `Procfile`, a text file that indicates which command should be used to start code running.  For our initial run we just want to initialize the database, so that's what goes in our `Procfile`:
    ```
    echo "web: airflow initdb" > Procfile
    ```

1. Commit once more and deploy to Heroku.  This will build the project on Heroku and run the database initialization command from the Procfile.  
    ```
    git add .
    git commit -m "Added configuration files."
    git push heroku master
    ```

1. Once deployed, follow the log output and await completion of the database initialization:
    ```
    heroku logs --tail
    ```

1. Now that the database is initialized, update `Procfile` to launch the web server:
    ```
    echo "web: airflow webserver --port \$PORT" > Procfile
    git add .
    git commit -m "Modify procfile to launch webserver"
    git push heroku master
    ```

1. Now when you launch the app (`heroku open`) there should be a logon screen.  
There is no logon yet, so we need to create a new user.  This can be done via Heroku bash (via [documentation](http://airflow.apache.org/security.html))
    ```
    heroku run bash
    python
    >>> import airflow
    >>> from airflow import models, settings
    >>> from airflow.contrib.auth.backends.password_auth import PasswordUser
    >>> user = PasswordUser(models.User())
    >>> user.username = 'new_user_name'
    >>> user.email = 'new_user_email@example.com'
    >>> user.password = 'set_the_password'
    >>> session = settings.Session()
    >>> session.add(user)
    >>> session.commit()
    >>> session.close()
    >>> exit()
    ```

1. Finally, modify the `Procfile` one last time to run both the web server and the scheduler.  
    ```
    echo "web:  airflow webserver --port \$PORT --daemon & airflow scheduler" > Procfile
    ```

1. Now any DAGs you want to run can go in a `dags` subfolder.  A great ETL example is here:  https://gtoonstra.github.io/etl-with-airflow/etlexample.html
