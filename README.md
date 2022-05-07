# Run Apache Airflow on Heroku
[Apache Airflow](https://airflow.apache.org/) can be used to create, schedule, and monitor workflows.  It is commonly used to define ETL processes.  An excellent example of an ETL workflow can be found [here](https://gtoonstra.github.io/etl-with-airflow/etlexample.html)

### Heroku Button deployment
Apache Airflow can be quickly and easily deployed to your own Heroku app by using this Heroku Button:
[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)

You will be prompted for a new Fernet key, which can be generated thusly:

    dd if=/dev/urandom bs=32 count=1 2>/dev/null | openssl base64

After deployment a login user will need to be created.  This can be done using the `create_user` command through Heroku bash ([documentation](https://airflow.apache.org/cli.html#create_user))
    
    heroku run bash
    airflow create_user -u <username> -p <password> -r <Role> -f <FirstName> -l <LastName> -e <Email>
    

### Manual Deployment
This is based largely on an excellent article ([here](https://medium.com/@damesavram/running-airflow-on-heroku-ed1d28f8013d)) on deploying Apache Airflow onto the Heroku platform, with some minor updates and tweaks.


1. Install or setup supported python version (I'm using [pyenv](https://github.com/pyenv/pyenv) so I just set the desired version in the project directory):
    ```
    echo "3.6.4" > .python-version
    ```
1. Create Python virtual environment to install Airflow along with dependencies
    ```
    python3 -m venv .venv
    source .venv/bin/activate
    ```

1. Install airflow, install cryptography module, and set Procfile to init db on initial run
    ```
    pip install "apache-airflow[postgres, password]"
    pip install "cryptography"
    pip freeze > requirements.txt
    ```

1. Create a `.gitignore` file
    ```
    echo ".venv/" > .gitignore
    ```

1. Initialize the git repository and create the Heroku app with a postgres add-on:
    ```
    git init
    git add .
    git commit -m "initial commit"

    heroku create
    heroku addons:create heroku-postgresql:hobby-dev
    ```

1. We will use `airflow.cfg` for most of our application configuration, but any secure values should be kept as Heroku config variables.  The `airflow.cfg` in this repository is already making use of the `DATABASE_URL` that was assigned when we created the database, but we will need a Fernet key in order to enable encryption for connection passwords stored in the database.  You can generate/set one thusly:
    ```
    heroku config:set AIRFLOW__CORE__FERNET_KEY=`dd if=/dev/urandom bs=32 count=1 2>/dev/null | openssl base64`
    ```
    We'll also need to set `AIRFLOW_HOME` to `/app` so that Airflow knows where the `airflow.cfg` file is.  Otherwise when the database initializes it will do so using sqlite, which on Heroku will only be created on an ephemeral file system that has the lifetime of the dyno running it:
    ```
    heroku config:set AIRFLOW_HOME=/app
    ```
    As SqlAlchemy no longer supports postgres://, we've had to add a new config var and change airflow.cfg to use it:
    ```
    heroku config:set NEW_DATABASE_URL=postgresql://bnjgfcozeqrkop:edcb2549f6871fce169205651b4cabd5614f65a9ef1e4dfe486a1fb06015aa96@ec2-52-48-159-67.eu-west-1.compute.amazonaws.com:5432/dd2kakv5hk6cdg

1. Heroku uses a `Procfile`, a text file that indicates which command should be used to start code running.  For our initial run we just want to initialize the database, so that's what goes in our `Procfile`:
    ```
    echo "web: airflow db init" > Procfile
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

1. Now when you launch the app (`heroku open`) there should be a logon screen.  There is no logon yet, so we need to create a new user.  This can be done using the `users create` command through Heroku bash ([documentation](https://airflow.apache.org/cli.html#create_user))
    ```
    heroku run bash
    airflow users create -u <username> -p <password> -r <Role> -f <FirstName> -l <LastName> -e <Email>
    ```

1. Finally, modify the `Procfile` one last time to run both the web server and the scheduler.  
    ```
    echo "web:  airflow webserver --port \$PORT --daemon & airflow scheduler" > Procfile
    ```

1. Any DAGs you want to run can go in a `dags` subfolder within the project.
