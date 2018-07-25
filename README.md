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
5. Update apt-get
    `$ sudo apt-get update`
6. Check no Python with `$ python -V` output: 
    The program 'python' can be found in the following packages:
    * python-minimal
    * python3
7. Install pip3 with `$ sudo apt-get install python3-pip`
8. Get virtualenv with `pip3 install virtualenv`
9. If `locale.Error: unsupported locale setting` error, run `export LC_ALL=C` from [here](http://bit.ly/2NK40lv)
10. Then run `pip3 install virtualenv` again
11. Create the project folder or clone the project from your Git repository
12. Set up the virtual environment.
    `$ cd cloned_project`
    `$ virtualanv py36 --python=python3.6` to create a Python 3.6 environment - Python 3.6 is required for altair
13. Activate the environment (use deactivate to exit the environment).
    `$ . py36/bin/activate`
14. Check python version with `$ python -V` output: `Python 3.6.6`
15. Install Flask and other required modules: (Note: You do not need to use pip3 or python3 within the virtualenv since it is already a Python3 virtualenv)
    `$ pip install Flask matplotlib numpy pandas altair bokeh holoviews`
16. Create the Flask file. In my case it exists already. => check that the file runs with `$ python grvapp.py` should get a dev server running with output: `* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)`
    press CTRL+C to quit
17. Setup Apache - need to build for the same version of Python
18. list available apache modeules `$ sudo apt-cache search libapache2*`
19. for Ubuntu, we need to use `$ sudo apt-get install apache2 apache2-dev` (from [mod_wsgi docs](http://bit.ly/2NKJ11N) and [apache docs](http://bit.ly/2NHZ0Oc))
20. From [mod_wsgi docs](http://bit.ly/2NKJ11N), Python must be 3.3 or later, we have 3.6.6 so *should be ok*.
21. Source code for mod_wsgi is [here](http://bit.ly/2NEPNX1), download the latests version and unpack it with `$ tar xvfz mod_wsgi-X.Y.tar.gz` replacing `X.Y` with version number. So for me: `$ tar xvfz mod_wsgi-4.6.4.tar.gz`
22. To setup the package ready for building run the “configure” script from within the source code directory:
    `./configure`
    Then build the source code with 
    `make`
    Then install the Apache module into the standard location for Apache modules as dictated by Apache for your installation, run:
    `make install` or `sudo make install` if permission denied
23. create a wsgi file with
    `$ cd ..` to come back into the correct folder and check paths:
    App path='/home/ubuntu/grv-app-deploy'
    virtualenv path='/home/ubuntu/grv-app-deploy/py36'
    html path='/var/www/html/grv-app-deploy'
    `$ vim app.wsgi`
24. Paste this code into the wsgi file:
    -------BEGIN-------
    activate_this = '/home/ubuntu/grv-app-deploy/py36/bin/activate_this.py'
    with open(activate_this) as f:
	    exec(f.read(), dict(__file__=activate_this))

    import sys
    import logging

    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/html/grv-app-deploy/")

    from grvapp import app as application
    -------END-------
25. Create a symlink so that the project directory appears in /var/www/html
    `$ sudo ln -sT ~/grv-app-deploy /var/www/html/grv-app-deploy`
26. Enable wsgi.
    `$ sudo a2enmod wsgi`
27. Configure apache (you will need to sudo to edit the file)
    `$ sudo vi /etc/apache2/sites-enabled/000-default.conf`
28. Paste this in right after the line with DocumentRoot /var/www/html
    ------BEGIN-------
    WSGIDaemonProcess grv-app-deploy threads=5
    WSGIScriptAlias / /var/www/html/grv-app-deploy/app.wsgi

    <Directory grv-app-deploy>
        WSGIProcessGroup grv-app-deploy
        WSGIApplicationGroup %{GLOBAL}
        Order deny,allow
        Allow from all
    </Directory>
    -------END-------
29. Restart the Server:
    `$ sudo apachectl restart`
30. check the IPv4 Public IP at e.g. `54.84.76.221`
    => returns `Internal Server Error`
31. check the logs at: `vim /var/log/apache2/error.log`
    => [wsgi:error] [pid 5719:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
