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
25. Create a symlink so that the project directory appears in /var/www/html
    `$ sudo ln -sT ~/grv-app-deploy /var/www/html/grv-app-deploy`
26. Enable wsgi.
    `$ sudo a2enmod wsgi`
27. Configure apache (you will need to sudo to edit the file)
    `$ sudo vi /etc/apache2/sites-enabled/000-default.conf`
28. Paste this in right after the line with DocumentRoot /var/www/html
    ```python
    WSGIDaemonProcess grv-app-deploy threads=5
    WSGIScriptAlias / /var/www/html/grv-app-deploy/app.wsgi

    <Directory grv-app-deploy>
        WSGIProcessGroup grv-app-deploy
        WSGIApplicationGroup %{GLOBAL}
        Order deny,allow
        Allow from all
    </Directory>
    ```
29. Restart the Server:
    `$ sudo apachectl restart`
30. check the IPv4 Public IP at e.g. `54.84.76.221`
    => returns `Internal Server Error`
31. check the logs at: `vim /var/log/apache2/error.log`

    => [wsgi:error] [pid 5719:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
    [apparently this can be ignored](http://bit.ly/2LIOSUP)

    Stack trace:

    ```
    [Wed Jul 25 07:11:34.348826 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00494: SIGHUP received.  Attempting to restart
    [Wed Jul 25 07:11:34.401663 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
    [Wed Jul 25 07:11:34.401681 2018] [core:notice] [pid 1283:tid 139903880587136] AH00094: Command line: '/usr/sbin/apache2'
    [Wed Jul 25 07:11:38.038388 2018] [wsgi:error] [pid 5719:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
    [Wed Jul 25 07:11:38.038419 2018] [wsgi:error] [pid 5719:tid 139903778035456]   return f(*args, **kwds)
    [Wed Jul 25 07:11:38.606178 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834] ERROR:flask.app:Exception on / [GET]
    [Wed Jul 25 07:11:38.606198 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834] Traceback (most recent call last):
    [Wed Jul 25 07:11:38.606203 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 2292, in wsgi_app
    [Wed Jul 25 07:11:38.606208 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     response = self.full_dispatch_request()
    [Wed Jul 25 07:11:38.606211 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1815, in full_dispatch_request
    [Wed Jul 25 07:11:38.606213 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     rv = self.handle_user_exception(e)
    [Wed Jul 25 07:11:38.606216 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1718, in handle_user_exception
    [Wed Jul 25 07:11:38.606218 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     reraise(exc_type, exc_value, tb)
    [Wed Jul 25 07:11:38.606221 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/_compat.py", line 35, in reraise
    [Wed Jul 25 07:11:38.606223 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     raise value
    [Wed Jul 25 07:11:38.606226 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1813, in full_dispatch_request
    [Wed Jul 25 07:11:38.606228 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     rv = self.dispatch_request()
    [Wed Jul 25 07:11:38.606230 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1799, in dispatch_request
    [Wed Jul 25 07:11:38.606234 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     return self.view_functions[rule.endpoint](**req.view_args)
    [Wed Jul 25 07:11:38.606236 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/var/www/html/grv-app-deploy/grvapp.py", line 22, in home
    [Wed Jul 25 07:11:38.606238 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     vol = pd.read_csv(volume_file, delim_whitespace=True)
    [Wed Jul 25 07:11:38.606240 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 678, in parser_f
    [Wed Jul 25 07:11:38.606243 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     return _read(filepath_or_buffer, kwds)
    [Wed Jul 25 07:11:38.606245 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 440, in _read
    [Wed Jul 25 07:11:38.606255 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     parser = TextFileReader(filepath_or_buffer, **kwds)
    [Wed Jul 25 07:11:38.606258 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 787, in __init__
    [Wed Jul 25 07:11:38.606260 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     self._make_engine(self.engine)
    [Wed Jul 25 07:11:38.606262 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 1014, in _make_engine
    [Wed Jul 25 07:11:38.606264 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     self._engine = CParserWrapper(self.f, **self.options)
    [Wed Jul 25 07:11:38.606266 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 1708, in __init__
    [Wed Jul 25 07:11:38.606269 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     self._reader = parsers.TextReader(src, **kwds)
    [Wed Jul 25 07:11:38.606272 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "pandas/_libs/parsers.pyx", line 384, in pandas._libs.parsers.TextReader.__cinit__
    [Wed Jul 25 07:11:38.606274 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "pandas/_libs/parsers.pyx", line 695, in pandas._libs.parsers.TextReader._setup_parser_source
    [Wed Jul 25 07:11:38.606278 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834] FileNotFoundError: File b'static/Volumes' does not exist
    [Wed Jul 25 07:11:38.606284 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]
    ```
32. to Fix this error, ensure the absolute path on the server is used: `/var/www/html/grv-app-deploy/static`
33. New error on `/altair` and `/entropy`:

    Stack trace:

    ```
    [Wed Jul 25 09:03:44.155676 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00494: SIGHUP received.  Attempting to restart
    [Wed Jul 25 09:03:44.208533 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
    [Wed Jul 25 09:03:44.208555 2018] [core:notice] [pid 1283:tid 139903880587136] AH00094: Command line: '/usr/sbin/apache2'
    [Wed Jul 25 09:03:52.476335 2018] [wsgi:error] [pid 8110:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
    [Wed Jul 25 09:03:52.476368 2018] [wsgi:error] [pid 8110:tid 139903778035456]   return f(*args, **kwds)
    [Wed Jul 25 09:03:54.629228 2018] [wsgi:error] [pid 8109:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
    [Wed Jul 25 09:03:54.629260 2018] [wsgi:error] [pid 8109:tid 139903778035456]   return f(*args, **kwds)
    [Wed Jul 25 09:03:55.211838 2018] [wsgi:error] [pid 8109:tid 139903778035456] [client 185.50.221.158:52480] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:55.334194 2018] [wsgi:error] [pid 8109:tid 139903635539712] [client 185.50.221.158:52480] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:55.457276 2018] [wsgi:error] [pid 8110:tid 139903769642752] [client 185.50.221.158:52483] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:55.578340 2018] [wsgi:error] [pid 8110:tid 139903761250048] [client 185.50.221.158:52483] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:55.697555 2018] [wsgi:error] [pid 8110:tid 139903752857344] [client 185.50.221.158:52483] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:56.929696 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] ERROR:flask.app:Exception on /altair [GET]
    [Wed Jul 25 09:03:56.929727 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] Traceback (most recent call last):
    [Wed Jul 25 09:03:56.929731 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/numpy/lib/npyio.py", line 440, in load
    [Wed Jul 25 09:03:56.929734 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     return pickle.load(fid, **pickle_kwargs)
    [Wed Jul 25 09:03:56.929737 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] _pickle.UnpicklingError: invalid load key, 'v'.
    [Wed Jul 25 09:03:56.929740 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]
    [Wed Jul 25 09:03:56.929743 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] During handling of the above exception, another exception occurred:
    [Wed Jul 25 09:03:56.929745 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]
    [Wed Jul 25 09:03:56.929759 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] Traceback (most recent call last):
    [Wed Jul 25 09:03:56.929762 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 2292, in wsgi_app
    [Wed Jul 25 09:03:56.929765 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     response = self.full_dispatch_request()
    [Wed Jul 25 09:03:56.929767 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1815, in full_dispatch_request
    [Wed Jul 25 09:03:56.929770 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     rv = self.handle_user_exception(e)
    [Wed Jul 25 09:03:56.929772 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1718, in handle_user_exception
    [Wed Jul 25 09:03:56.929783 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     reraise(exc_type, exc_value, tb)
    [Wed Jul 25 09:03:56.929786 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/_compat.py", line 35, in reraise
    [Wed Jul 25 09:03:56.929795 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     raise value
    [Wed Jul 25 09:03:56.929798 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1813, in full_dispatch_request
    [Wed Jul 25 09:03:56.929800 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     rv = self.dispatch_request()
    [Wed Jul 25 09:03:56.929803 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1799, in dispatch_request
    [Wed Jul 25 09:03:56.929805 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     return self.view_functions[rule.endpoint](**req.view_args)
    [Wed Jul 25 09:03:56.929807 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/var/www/html/grv-app-deploy/grvapp.py", line 109, in altair
    [Wed Jul 25 09:03:56.929810 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     mid_unit = np.load('/var/www/html/grv-app-deploy/static/mid_unit.npy')
    [Wed Jul 25 09:03:56.929812 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/numpy/lib/npyio.py", line 443, in load
    [Wed Jul 25 09:03:56.929815 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     "Failed to interpret file %s as a pickle" % repr(file))
    [Wed Jul 25 09:03:56.929819 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] OSError: Failed to interpret file '/var/www/html/grv-app-deploy/static/mid_unit.npy' as a pickle
    [Wed Jul 25 09:03:56.929825 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]
    ```
34. it appears the large files are not saved on the ubuntu server, if this is the case it would explain the error above, so attempt to use [Git LFS](https://git-lfs.github.com/) as it was used to upload them to github to try to download them:
    follow [this path](https://github.com/git-lfs/git-lfs/wiki/Installation)
    `$ curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash`
    then:
    `$ sudo apt-get install git-lfs`
    and:
    `$ git lfs install`
35. then `$ git lfs pull` to download large files.
36. This worked, but now the page "didn't send any data", perhaps due to large file size
37. use `$ sudo bash -c 'echo > /var/log/apache2/error.log'` to clean error log and `$ sudo apachectl restart` to restart the server
38. While the cause of the 'page didn't send any data' is not clear, Diego Castañeda suggested [mounting the S3 bucket onto the EC2 instance](http://bit.ly/2NIVgMl)
39. Procedure followed to set up an [S3 bucket](https://amzn.to/2LTKDGd)
    - this was initially done directly with the numpy pickles, but as this still does not work...
    - next step is to try with a small image to check the S3 connection is working, 
    - then a large image, to check the file size is not an issue
40. Checking with a small image:
    - upload small image to S3 - done, ok
    - check data available on EC2-mounted S3 bucket - done, ok
    - create new route in app - done, ok
    - test1: page loads with alt-text working correctly. no image but `Distribution` still `In Progress`
    - test2: now `Distribution` is `Available`, reload page: working, image is seen from the `Distribution`.
41. Check with a large image:
    - upload small image to S3 -
    - check data available on EC2-mounted S3 bucket -
    - create new route in app -
    - test1: page loads with alt-text working correctly. no image but `Distribution` still `In Progress`
    - test2: now `Distribution` is `Available`, reload page:


## Change ownership and permissions on bucket

Run:
- `$ sudo chown -R www-data:www-data /var/www/html/grv-app-deploy/s3bucket/`
- `$ sudo chmod -R 775 /var/www/html/grv-app-deploy/s3bucket/`


