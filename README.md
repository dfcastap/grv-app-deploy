# grv-app-deploy

This repo is a self-contained copy of the working app.

It works locally, the aim is to deploy it on Amazon EC2

## Procedure

This procedure is based on [this blog](http://bit.ly/2LJUONn) and its help is gratefully acknowledged

1. Sign in to AWS services

2. Launch EC2 instance using `Canonical, Ubuntu, 16.04 LTS, amd64 xenial image build on 2018-06-27`

3. Connect to the instance using SSH where permissions on `key.pem` were set with
    `$ chmod 400 key.pem`

4. Connect with `ssh -i ~/.ssh/grv-app-deploy.pem ubuntu@ec2-54-84-76-221.compute-1.amazonaws.com`

5. Update apt
    `$ sudo apt update`

6. **Always make sure all packages are up to date too**

    **`$ sudo apt upgrade`**

7. Check no Python with `$ python -V` output: 
    The program 'python' can be found in the following packages:
    * python-minimal
    * python3

8. Install python 3.6 with:
    * `sudo add-apt-repository ppa:deadsnakes/ppa`
    * `sudo apt-get update`
    * `sudo apt-get install python3.6 python3.6-dev`

9. Install pip3 with `$ sudo apt-get install python3-pip`

10. Get virtualenv with `pip3 install virtualenv`

11. If `locale.Error: unsupported locale setting` error, run `export LC_ALL=C` from [here](http://bit.ly/2NK40lv)

12. Then run `pip3 install virtualenv` again

13. Create the project folder or clone the project from your Git repository

14. Set up the virtual environment.
    `$ cd cloned_project`
    `$ virtualenv py36 --python=python3.6` to create a Python 3.6 environment - Python 3.6 is required for altair

15. Activate the environment (use deactivate to exit the environment).
    `$ . py36/bin/activate`

16. Check python version with `$ python -V` output: `Python 3.6.6`

17. Install Flask and other required modules: (Note: You do not need to use pip3 or python3 within the virtualenv since it is already a Python3 virtualenv)
    **`$ pip install Flask matplotlib numpy pandas altair bokeh "holoviews[recommended]"`**

18. Create the Flask file. In my case it exists already. => check that the file runs with `$ python grvapp.py` should get a dev server running with output: `* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)`
    press CTRL+C to quit

19. Setup Apache - need to build for the same version of Python

20. list available apache modeules `$ sudo apt-cache search libapache2*`

21. for Ubuntu, we need to use `$ sudo apt-get install apache2 apache2-dev` (from [mod_wsgi docs](http://bit.ly/2NKJ11N) and [apache docs](http://bit.ly/2NHZ0Oc))

22. From [mod_wsgi docs](http://bit.ly/2NKJ11N), Python must be 3.3 or later, we have 3.6.6 so *should be ok*.

23. Source code for mod_wsgi is [here](http://bit.ly/2NEPNX1), download the latests version and unpack it with `$ tar xvfz mod_wsgi-X.Y.tar.gz` replacing `X.Y` with version number. So for me: `$ tar xvfz mod_wsgi-4.6.4.tar.gz`

24. To setup the package ready for building run the “configure” script from within the source code directory:
    `./configure`
    Then build the source code with 
    `make`
    Then install the Apache module into the standard location for Apache modules as dictated by Apache for your installation, run:
    **~~`make install` or~~ `sudo make install` ~~if permission denied~~**

25. **`sudo make install` will probably not install the `mod_wsgi` python library in the right place. With the python 3.6 env activated and inside the mod_wsgi install dir, do:**

    **`python setup.py install`**

26. create a wsgi file with
    `$ cd ..` to come back into the correct folder and check paths:
    App path='/home/ubuntu/grv-app-deploy'
    virtualenv path='/home/ubuntu/grv-app-deploy/py36'
    html path='/var/www/html/grv-app-deploy'
    `$ vim app.wsgi`

27. Paste this code into the wsgi file:
    ```python
    activate_this = '/home/ubuntu/grv-app-deploy/py36/bin/activate_this.py'
    with open(activate_this) as f:
        exec(f.read(), dict(__file__=activate_this))
    
    import sys
    import logging
    
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/html/grv-app-deploy/")
    
    from grvapp import app as application
    ```

28. Create a symlink so that the project directory appears in /var/www/html
    `$ sudo ln -sT ~/grv-app-deploy /var/www/html/grv-app-deploy`

29. **In my case, I had to add the apache config file that would let apache load the wsgi module:**

    **`sudo vi /etc/apache2/mods-available/wsgi.load`**

    **include in that file the following line:**

    **`LoadModule wsgi_module /usr/lib/apache2/modules/mod_wsgi.so`**

30. Enable wsgi.
    `$ sudo a2enmod wsgi`

31. Configure apache (you will need to sudo to edit the file)
    `$ sudo vim /etc/apache2/sites-enabled/000-default.conf`

32. Paste this in right after the line with DocumentRoot /var/www/html
    ```python
    WSGIDaemonProcess grv-app-deploy threads=5 socket-timeout=20 memory-limit=850000000 virtual-memory-limit=850000000
    WSGIScriptAlias / /var/www/html/grv-app-deploy/app.wsgi
    
    <Directory grv-app-deploy>
        WSGIProcessGroup grv-app-deploy
        WSGIApplicationGroup %{GLOBAL}
        Order deny,allow
        Allow from all
    </Directory>
    ```

33. Restart the Server:
    `$ sudo apachectl restart`

34. Ensure that all path references in the app are absolute: `/var/www/html/grv-app-deploy/static`


## Change ownership and permissions on app folder

Run:
- `$ sudo chown -R www-data:www-data /var/www/html/`

  

**Always double check inside `grv-app-deploy` and subdirectories that AT LEAST, the group of every file is `www-data` and that group members have `rwx` permission.**